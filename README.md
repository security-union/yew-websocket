![Ww0Xt1mAMy31Ofar0GYu8Oab0v2k0uF1XT_zTt5kPU1M8o58sT5OOXCsSxv3nNGxsG8dG4zI=w1060-fcrop64=1,00005a57ffffa5a8-k-c0xffffffff-no-nd-rj (1)](https://user-images.githubusercontent.com/1176339/155262320-ce1406f0-d35d-418e-a8b9-60b928cceeb2.jpeg)

# yew-websocket
[![crates.io](https://img.shields.io/crates/v/yew-websocket.svg)](https://crates.io/crates/yew-websocket)
[![docs.rs](https://docs.rs/yew-websocket/badge.svg)](https://docs.rs/yew-websocket)

Rust yew websocket service written with love :)

This crate is based on the original yew websocket service that used to be part of the core library.
https://github.com/yewstack/yew/blob/0.18.0/packages/yew/src/services/websocket.rs

For some reason, the core team decided to kill it.

I tried using the suggested libraries (wasm-sockets or gloo-net), but those are not properly integrated with yew.

## Sample

```rust
use anyhow::Error;
use serde_derive::{Deserialize, Serialize};
use yew_websocket::macros::Json;

use yew::{html, Component, Context, Html};
use yew_websocket::websocket::{WebSocketService, WebSocketStatus, WebSocketTask};

type AsBinary = bool;

pub enum Format {
    Json,
    Toml,
}

pub enum WsAction {
    Connect,
    SendData(AsBinary),
    Disconnect,
    Lost,
}

pub enum Msg {
    WsAction(WsAction),
    WsReady(Result<WsResponse, Error>),
}

impl From<WsAction> for Msg {
    fn from(action: WsAction) -> Self {
        Msg::WsAction(action)
    }
}

/// This type is used as a request which sent to websocket connection.
#[derive(Serialize, Debug)]
struct WsRequest {
    value: u32,
}

/// This type is an expected response from a websocket connection.
#[derive(Deserialize, Debug)]
pub struct WsResponse {
    value: u32,
}

pub struct Model {
    pub fetching: bool,
    pub data: Option<u32>,
    pub ws: Option<WebSocketTask>,
}

impl Model {
    fn view_data(&self) -> Html {
        if let Some(value) = self.data {
            html! {
                <p>{ value }</p>
            }
        } else {
            html! {
                <p>{ "Data hasn't fetched yet." }</p>
            }
        }
    }
}

impl Component for Model {
    type Message = Msg;
    type Properties = ();

    fn create(ctx: &Context<Self>) -> Self {
        Self {
            fetching: false,
            data: None,
            ws: None,
        }
    }

    fn update(&mut self, ctx: &Context<Self>, msg: Self::Message) -> bool {
        match msg {
            Msg::WsAction(action) => match action {
                WsAction::Connect => {
                    let callback = ctx.link().callback(|Json(data)| Msg::WsReady(data));
                    let notification = ctx.link().batch_callback(|status| match status {
                        WebSocketStatus::Opened => None,
                        WebSocketStatus::Closed | WebSocketStatus::Error => {
                            Some(WsAction::Lost.into())
                        }
                    });
                    let task = WebSocketService::connect(
                        "wss://echo.websocket.events/",
                        callback,
                        notification,
                    )
                    .unwrap();
                    self.ws = Some(task);
                    true
                }
                WsAction::SendData(binary) => {
                    let request = WsRequest { value: 321 };
                    if binary {
                        self.ws.as_mut().unwrap().send_binary(Json(&request));
                    } else {
                        self.ws.as_mut().unwrap().send(Json(&request));
                    }
                    false
                }
                WsAction::Disconnect => {
                    self.ws.take();
                    true
                }
                WsAction::Lost => {
                    self.ws = None;
                    true
                }
            },
            Msg::WsReady(response) => {
                self.data = response.map(|data| data.value).ok();
                true
            }
        }
    }

    fn view(&self, ctx: &Context<Self>) -> Html {
        html! {
            <div>
                <nav class="menu">
                    { self.view_data() }
                    <button disabled={self.ws.is_some()}
                            onclick={ctx.link().callback(|_| WsAction::Connect)}>
                        { "Connect To WebSocket" }
                    </button>
                    <button disabled={self.ws.is_none()}
                            onclick={ctx.link().callback(|_| WsAction::SendData(false))}>
                        { "Send To WebSocket" }
                    </button>
                    <button disabled={self.ws.is_none()}
                            onclick={ctx.link().callback(|_| WsAction::SendData(true))}>
                        { "Send To WebSocket [binary]" }
                    </button>
                    <button disabled={self.ws.is_none()}
                            onclick={ctx.link().callback(|_| WsAction::Disconnect)}>
                        { "Close WebSocket connection" }
                    </button>
                </nav>
            </div>
        }
    }
}

fn main() {
    yew::start_app::<Model>();
}
```