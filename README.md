This is a guide on how to build a leptos tauri project from scratch without using a template.
<br><br>
We've already done the following
```sh
cargo new leptos_tauri_from_scratch
```

First, make our two seperate project folders. We need one for our actual app, 'src-orig' and the other is required when using `cargo tauri`
```sh
mkdir src-orig && mkdir src-tauri
```

Let's delete the original src folder,
```sh
rm -r src
```
 and rewrite the `Cargo.toml` file to the following.
```toml
[workspace]
resolver = "2"
members = ["src-tauri", "src-orig"]

[profile.release]
codegen-units = 1
lto = true
```
We'll use resolver two because we're using a modern version of Rust. We'll list our workspace members. `codegen-units = 1` and `lto = true` are good things to have for our eventual release.
<br><br>
What we're going to want to do is use cargo leptos for building our SSR server and trunk for building our CSR client that we bundle into apps with tauri.
Let's add a `Trunk.toml` file.
```toml
[build]
target = "./src-orig/index.html"

[watch]
ignore = ["./src-tauri"]
```

The target is the index.html that trunk uses to build the wasm and js files that we'll need for the bundling process when we call `cargo tauri build`
<br>
Let's create the `index.html` file

```sh
touch src-orig/index.html
```

Let's fill it with
```html
<!DOCTYPE html>
<html>
	<head>
		<link data-trunk rel="rust" data-wasm-opt="z" data-bin="leptos_tauri_from_scratch_bin"/>
		<link rel="icon" type="image/x-icon" href="favicon.ico">
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
	</head>
	<body></body>
</html>
```
This line
```html
<link data-trunk rel="rust" data-wasm-opt="z" data-bin="leptos_tauri_from_scratch_bin"/>
```
Tells trunk we want to use compile our wasm to be small `opt="z"` and that our binary will be named `"leptos_tauri_from_scratch_bin"`. We need to specify a that our binary will be a different name then our project name because we are also going to library wasm file and if we don't differentiate here then `cargo tauri` will get confused. More specifically two artifacts will be generated, one for the lib and the other for the binary and it won't know which to use.
<br><br>
Let's create a favicon that we referenced.
```
mkdir public && curl https://raw.githubusercontent.com/leptos-rs/leptos/main/examples/animated_show/public/favicon.ico > public/favicon.ico
```
The other parts of the html, are standard setting a charset to utf-8 and the viewport content will be helpful when building for mobile.
<br><br>
Let's create a tauri configuration file.
```sh
touch src-tauri/taur.conf.json
```
And drop this in there
```json
{
  "build": {
    "beforeDevCommand": "",
    "beforeBuildCommand": "trunk build --no-default-features -v --features \"csr\"",
    "devPath": "http://127.0.0.1:3000",
    "distDir": "../src-orig/dist"
  },
  "package": {
    "productName": "leptos_tauri_from_scratch",
    "version": "0.1.0"
  },
  "tauri": {
    "windows": [
      {
        "fullscreen": false,
        "height": 800,
        "resizable": true,
        "title": "LeptosChatApp",
        "width": 1200
      }
    ],
    "bundle": {
      "active": true,
      "category": "DeveloperTool",
      "copyright": "",
      "deb": {
        "depends": []
      },
      "externalBin": [],
      "icon": [],
      "identifier": "leptos.chat.app",
      "longDescription": "",
      "macOS": {
        "entitlements": null,
        "exceptionDomain": "",
        "frameworks": [],
        "providerShortName": null,
        "signingIdentity": null
      },
      "resources": [],
      "shortDescription": "",
      "targets": "all",
      "windows": {
        "certificateThumbprint": null,
        "digestAlgorithm": "sha256",
        "timestampUrl": ""
      }
    },
    "security": {
      "csp": null
    }
  }
}
```
You can basically ignore all of this except for
```json
  "build": {
    "beforeDevCommand": "",
    "beforeBuildCommand": "trunk build --no-default-features -v --features \"csr\"",
    "devPath": "http://127.0.0.1:3000",
    "distDir": "../src-orig/dist"
  },
  "package": {
    "productName": "leptos_tauri_from_scratch",
    "version": "0.1.0"
  },
  "tauri": {
    "windows": [
      {
        "fullscreen": false,
        "height": 800,
        "resizable": true,
        "title": "LeptosChatApp",
        "width": 1200
      }
    ],
```
Let's look at 
```json
    "beforeBuildCommand": "trunk build --no-default-features -v --features \"csr\"",
```
When we write `cargo tauri build` this will run before hand. Trunk will run it's build process, using the index.html file in the src-orig that we specified in `Trunk.toml` We'll run only with the CSR feature. This is important. We are going to build an SSR app, and serve it over the internet but we are going to build a tauri client for desktop and mobile using CSR and then it's going to make network requests to our server that is servering our app to browsers using SSR. This is the best of both worlds, we get the SEO of SSR and other advantages while being able to use CSR to build our app for other platforms.
```
    "devPath": "http://127.0.0.1:3000",
    "distDir": "../src-orig/dist"
```
Check https://tauri.app/v1/api/config/#buildconfig for what these do, they aren't very relevant in this guide.
<br><br>
This is just how big we want the window.
```
    "windows": [
      {
        "fullscreen": false,
        "height": 800,
        "resizable": true,
        "title": "LeptosChatApp",
        "width": 1200
      }
    ],
```
Let's add a Cargo.toml to both of our packages.
```sh
touch src-tauri/Cargo.toml
```
Let's change that file to this, we're using the 2.0.0 alpha version of tauri to build to mobile.
```toml
[package]
name = "src_tauri"
version = "0.0.1"
edition = "2021"

[lib]
name="app_lib"
path="src/lib.rs"

[build-dependencies]
tauri-build = { version = "2.0.0-alpha.13", features = [] }

[dependencies]
log = "0.4.0"
serde = { version = "1.0", features = ["derive"] }
tauri = { version = "2.0.0-alpha.20", features = ["devtools"] }
tauri-plugin-http = "2.0.0-alpha.9"

[features]
#default = ["custom-protocol"]
custom-protocol = ["tauri/custom-protocol"]
```
To make use of `cargo tauri build` we need `tauri-build` and we also need a `build.rs`
```
touch src-tauri/build.rs
```
And let's change that to
```
fn main() {
    tauri_build::build();
}
```
And never think about it again.<br><br>
Let's create the cargo.toml for our leptos app.
```
touch src-orig/Cargo.toml
```
And let's dump in
```
[package]
name = "leptos_tauri_from_scratch"
version = "0.1.0"
edition = "2021"


[lib]
crate-type = ["staticlib", "cdylib", "rlib"]

[[bin]]
name="leptos_tauri_from_scratch_bin"
path="./src/main.rs"

[dependencies]
axum = {version = "0.7.0", optional=true}
axum-macros = { version= "0.4.1", optional=true}
cfg-if = "1.0.0"
console_error_panic_hook = "0.1.7"
console_log = "1.0.0"
leptos = { git = "https://github.com/leptos-rs/leptos.git", branch = "leptos_v0.6" }
leptos-use = "0.9.0"
leptos_axum = { git = "https://github.com/leptos-rs/leptos.git", branch = "leptos_v0.6", optional = true }
leptos_meta = { git = "https://github.com/leptos-rs/leptos.git", branch = "leptos_v0.6" }
leptos_router = { git = "https://github.com/leptos-rs/leptos.git", branch = "leptos_v0.6" }
log = "0.4.20"
serde = "1.0.195"
serde_json = "1.0.111"
server_fn = { git = "https://github.com/leptos-rs/leptos.git", branch = "leptos_v0.6" }
tokio = { version = "1.35.1", optional=true }
tower = {version = "0.4.10", optional = true}
tower-http = { version = "0.5.1", optional = true, features= ["fs","cors"] }
wasm-bindgen = "0.2.89"

[features]
csr = [ "leptos/csr","leptos_meta/csr","leptos_router/csr", ]
hydrate = ["leptos/hydrate", "leptos_meta/hydrate", "leptos_router/hydrate"]
ssr = [
    "dep:axum",
    "dep:axum-macros",
    "leptos/ssr",
    "leptos-use/ssr",
    "dep:leptos_axum",
    "leptos_meta/ssr",
    "leptos_router/ssr",
    "dep:tower-http",
    "dep:tower",
    "dep:tokio",
]   

[package.metadata.leptos]
bin-exe-name="leptos_tauri_from_scratch_bin"
output-name="leptos_tauri_from_scratch"
assets-dir = "../public"
site-pkg-dir = "pkg"
site-root = "target/site"
site-addr = "0.0.0.0:3000"
reload-port = 3001
browserquery = "defaults"
watch = false
env = "DEV"
bin-features = ["ssr"]
bin-default-features = false
lib-features = ["hydrate"]
lib-default-features = false
```
So this looks like a normal SSR leptos, except for<br>
- We have a CSR, Hydrate, and SSR versions.
```toml
[features]
csr = [ "leptos/csr","leptos_meta/csr","leptos_router/csr", ]
hydrate = ["leptos/hydrate", "leptos_meta/hydrate", "leptos_router/hydrate"]
ssr = [
```
also our binary is specified and named
```toml
[[bin]]
name="leptos_tauri_from_scratch_bin"
path="./src/main.rs"
```
our lib is specified, but unnamed (it will default to the project name in cargo leptos and in cargo tauri)
```toml
[lib]
crate-type = ["staticlib", "cdylib", "rlib"]
```

We've added the override to our cargo leptos metadata.
```toml
[package.metadata.leptos]
bin-exe-name="leptos_tauri_from_scratch_bin"
```
Our tauri app is going to send server function calls to this address, this is where we'll server our hydratable SSR client from.
```
site-addr = "0.0.0.0:3000"
```
Now let's create the `main.rs` that we reference in the `src-orig/Cargo.toml`
```
mkdir src-orig/src && touch src-orig/src/main.rs
```
and drop this in there...
```rust
cfg_if::cfg_if! {
    if #[cfg(feature="ssr")] {
        use tower_http::cors::{CorsLayer};
        use axum::{
            Router,
            routing::get,
            extract::State,
            http::Request,
            body::Body,
            response::IntoResponse
        };
        use leptos::{*,provide_context, LeptosOptions};
        use leptos_axum::LeptosRoutes;
        use leptos_tauri_from_scratch::fallback::file_and_error_handler;

        #[derive(Clone,Debug,axum_macros::FromRef)]
        pub struct ServerState{
            pub options:LeptosOptions,
            pub routes: Vec<leptos_router::RouteListing>,
        }

        pub async fn server_fn_handler(
            State(state): State<ServerState>,
            request: Request<Body>,
        ) -> impl IntoResponse {
            leptos_axum::handle_server_fns_with_context(
                move || {
                    provide_context(state.clone());
                },
                request,
            )
            .await
            .into_response()
        }

        pub async fn leptos_routes_handler(
            State(state): State<ServerState>,
            req: Request<Body>,
        ) -> axum::response::Response {
            let handler = leptos_axum::render_route_with_context(
                state.options.clone(),
                state.routes.clone(),
                move || {
                    provide_context("...");
                },
                leptos_tauri_from_scratch::App,
            );
            handler(req).await.into_response()
        }

        #[tokio::main]
        async fn main() {
            let conf = get_configuration(Some("./src-orig/Cargo.toml")).await.unwrap();

            let leptos_options = conf.leptos_options;
            let addr = leptos_options.site_addr;
            let routes =  leptos_axum::generate_route_list(leptos_tauri_from_scratch::App);

            let state = ServerState{
                options:leptos_options,
                routes:routes.clone(),
            };

            let cors = CorsLayer::new()
                .allow_methods([axum::http::Method::GET, axum::http::Method::POST])
                .allow_origin("tauri://localhost".parse::<axum::http::HeaderValue>().unwrap())
                .allow_headers(vec![axum::http::header::CONTENT_TYPE]);


            let app = Router::new()
                .route("/api/*fn_name",get(server_fn_handler).post(server_fn_handler))
                .layer(cors)
                .leptos_routes_with_handler(routes, get(leptos_routes_handler))
                .fallback(file_and_error_handler)
                .with_state(state);

            let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
            logging::log!("listening on http://{}", &addr);
                axum::serve(listener, app.into_make_service())
                    .await
                    .unwrap();
        }
    } else if #[cfg(feature="csr")]{
        pub fn main() {
            server_fn::client::set_server_url("http://127.0.0.1:3000");
            leptos::mount_to_body(leptos_tauri_from_scratch::App);
        }
    } else {
        pub fn main() {

        }
    }
}
```
This is our three pronged binary.
When we run cargo leptos server, we're going to get a server that is what's in our `if #[cfg(feature="ssr")] {` branch. We're going to hydrate, that's final `else` branch that is just empty. That actually gets filled in or something with a call to hydrate.
<br>
And our csr feature 
```
 else if #[cfg(feature="csr")]{
        pub fn main() {
            server_fn::client::set_server_url("http://127.0.0.1:3000");
            leptos::mount_to_body(leptos_tauri_from_scratch::App);
        }
    }
```
Here we're setting the server functions to use the url base we've input here. I.e local host, on the port we specified in the leptos metadata.<br>
Otherwise our tauri all will try to route server function network requests using it's own idea of what it's url is. Which is like `tauri://localhost` on macOs or something!
<br>
Since we are going to be getting API requests from different locations beside our server's domain let's set up CORs, if you don't do this your tauri apps won't be able to make server function calls.
```
            let cors = CorsLayer::new()
                .allow_methods([axum::http::Method::GET, axum::http::Method::POST])
                .allow_origin("tauri://localhost".parse::<axum::http::HeaderValue>().unwrap())
                .allow_headers(vec![axum::http::header::CONTENT_TYPE]);
```
And just layer it.
```
                .layer(cors)
```
If you are on windows the origin of your app will be different then "tauri://localhost" and you'll need to figure that out, as well as if you deploy it to places that aren't your localhost!
<br>
Everything else is standard leptos, so let's fill in the fallback and the lib really quick.
```
touch src-orig/src/lib.rs && touch src-orig/src/fallback.rs
```

Let's dump this bog standard leptos code in the lib.rs
```rust
use leptos::*;

#[cfg(feature = "ssr")]
pub mod fallback;

#[server(endpoint = "hello_world")]
pub async fn hello_world_server() -> Result<String, ServerFnError> {
    Ok("Hey.".to_string())
}

#[component]
pub fn App() -> impl IntoView {
    let action = create_server_action::<HelloWorldServer>();
    let vals = create_rw_signal(String::new());
    create_effect(move |_| {
        if let Some(resp) = action.value().get() {
            match resp {
                Ok(val) => vals.set(val),
                Err(err) => vals.set(format!("{err:?}")),
            }
        }
    });
    view! {<button
        on:click=move |_| {
            action.dispatch(HelloWorldServer{});
        }
        >"Hello world."</button>
        {
            move || vals.get()
        }
    }
}
cfg_if::cfg_if! {
    if #[cfg(feature = "hydrate")] {
        use wasm_bindgen::prelude::wasm_bindgen;

        #[wasm_bindgen]
        pub fn hydrate() {
            #[cfg(debug_assertions)]
            console_error_panic_hook::set_once();
            leptos::mount_to_body(App);
        }
    }
}
```
and add a generic fallback file.
```rust
use axum::{
    body::Body,
    extract::State,
    http::{Request, Response, StatusCode, Uri},
    response::{IntoResponse, Response as AxumResponse},
};
use leptos::{view, LeptosOptions};
use tower::ServiceExt;
use tower_http::services::ServeDir;

pub async fn file_and_error_handler(
    uri: Uri,
    State(options): State<LeptosOptions>,
    req: Request<Body>,
) -> AxumResponse {
    let root = options.site_root.clone();
    let res = get_static_file(uri.clone(), &root).await.unwrap();

    if res.status() == StatusCode::OK {
        res.into_response()
    } else {
        let handler = leptos_axum::render_app_to_stream(options.to_owned(), move || view! {404});
        handler(req).await.into_response()
    }
}

async fn get_static_file(uri: Uri, root: &str) -> Result<Response<Body>, (StatusCode, String)> {
    let req = Request::builder()
        .uri(uri.clone())
        .body(Body::empty())
        .unwrap();
    match ServeDir::new(root).oneshot(req).await {
        Ok(res) => Ok(res.into_response()),
        Err(err) => Err((
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {err}"),
        )),
    }
}
```
Let's fill in our src-tauri/src folder.
```
mkdir src-tauri/src && touch src-tauri/src/main.rs && touch src-tauri/src/lib.rs
```
and drop this in `main.rs`
```rust
// Prevents additional console window on Windows in release, DO NOT REMOVE!!
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    app_lib::run();
}
```
and in `lib.rs`
```
use tauri::Manager;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_http::init())
        .setup(|app| {
            {
                let window = app.get_window("main").unwrap();
                window.open_devtools();
            }
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```
We're gonna open devtools right away to see what is going on in our app. We need the tauri_http_plugin to make http calls, and generate_context reads our `tauri.conf.json` file.
<br><br>
We need an icon folder and an icon to build.
```
mkdir src-tauri/icons && curl https://raw.githubusercontent.com/tauri-apps/tauri/dev/examples/.icons/128x128.png > src-tauri/icons/icon.png
```
set nightly
```sh
rustup override set nightly
```
Then run 
```sh
cargo leptos serve
```
You should get
```sh
➜  lepto_tauri_from_scratch git:(main) ✗ cargo leptos serve
    Finished dev [unoptimized + debuginfo] target(s) in 0.60s
       Cargo finished cargo build --package=leptos_tauri_from_scratch --lib --target-dir=/Users/sam/Projects/lepto_tauri_from_scratch/target/front --target=wasm32-unknown-unknown --no-default-features --features=hydrate
       Front compiling WASM
    Finished dev [unoptimized + debuginfo] target(s) in 0.93s
       Cargo finished cargo build --package=leptos_tauri_from_scratch --bin=leptos_tauri_from_scratch_bin --no-default-features --features=ssr
     Serving at http://0.0.0.0:3000
listening on http://0.0.0.0:3000
```
Now open a new terminal and
```sh
cargo tauri build
```
It'll build with csr before
```sh
Running beforeBuildCommand `trunk build --no-default-features -v --features "csr"`
```
and then you should have your app, I'm on macOs so here's what I get. It's for desktop.
```
 Compiling src_tauri v0.0.1 (/Users/sam/Projects/lepto_tauri_from_scratch/src-tauri)
    Finished release [optimized] target(s) in 2m 26s
    Bundling leptos_tauri_from_scratch.app (/Users/sam/Projects/lepto_tauri_from_scratch/target/release/bundle/macos/leptos_tauri_from_scratch.app)
    Bundling leptos_tauri_from_scratch_0.1.0_x64.dmg (/Users/sam/Projects/lepto_tauri_from_scratch/target/release/bundle/dmg/leptos_tauri_from_scratch_0.1.0_x64.dmg)
    Running bundle_dmg.sh
```
Open run it and voilá.