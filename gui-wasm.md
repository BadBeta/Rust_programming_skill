# GUI, Web Frontend, and WebAssembly

Desktop GUI with egui and iced, web frontend with Leptos, Yew, and Dioxus, and WebAssembly for browser and server-side execution.

## Rules for GUI, Web Frontend & WASM (LLM)

1. **ALWAYS batch JS↔WASM boundary crossings** — each crossing has overhead; pass arrays/buffers instead of calling per-element functions
2. **ALWAYS enable exactly one render mode** for Leptos (`csr`, `hydrate`, or `ssr`) — the build fails silently or produces wrong output with multiple or zero modes enabled
3. **ALWAYS use `signal()` not `signal()`** in Leptos 0.7+ — `signal` is deprecated; `signal()` returns `(ReadSignal, WriteSignal)` that are `Copy`
4. **PREFER Leptos for new full-stack web apps** — first-class SSR, server functions, smaller bundle; use Yew only for SPA-only apps with mature ecosystem needs
5. **ALWAYS use `closure.forget()` for JS event listeners** in wasm-bindgen — if the `Closure` is dropped, the listener silently stops working with no error
6. **ALWAYS use `opt-level = "z"` and `lto = true`** in WASM release profiles — WASM binary size directly impacts load time; also use `wasm-opt -Os` post-build
7. **PREFER egui for tools and prototypes, iced for production apps** — egui is faster to build (immediate mode) but harder to style; iced provides structured state management
8. **NEVER use `wasm-pack`** — archived January 2025; use Trunk for web apps or `wasm-bindgen-cli` for libraries

### Common Mistakes (BAD/GOOD)

**Deprecated Leptos signals:**
```rust
// BAD: signal is deprecated in Leptos 0.7+
let (count, set_count) = signal(0);

// GOOD: use signal() — returns Copy types
let (count, set_count) = signal(0);
```

**Many small WASM boundary crossings:**
```rust
// BAD: 1000 boundary crossings
#[wasm_bindgen]
pub fn process_one(x: f64) -> f64 { x * 2.0 }
// JS: for (let i = 0; i < 1000; i++) process_one(arr[i]);

// GOOD: single boundary crossing with batch
#[wasm_bindgen]
pub fn process_batch(data: &[f64]) -> Vec<f64> {
    data.iter().map(|x| x * 2.0).collect()
}
```

**Dropping Closure for event listeners:**
```rust
// BAD: closure dropped immediately — event listener silently stops
let closure = Closure::wrap(Box::new(|_: web_sys::MouseEvent| {
    web_sys::console::log_1(&"clicked".into());
}) as Box<dyn FnMut(_)>);
element.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
// closure dropped here! listener broken!

// GOOD: forget() keeps the closure alive (or store it in struct)
closure.forget();
```

## Desktop GUI — egui (Immediate Mode)

egui redraws the entire UI every frame. State lives in your struct; no message passing needed.

### Basic Application

```rust
// Cargo.toml: eframe = "0.28"
use eframe::egui;

fn main() -> eframe::Result<()> {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default().with_inner_size([400.0, 300.0]),
        ..Default::default()
    };
    eframe::run_native("My App", options, Box::new(|_cc| Ok(Box::new(MyApp::default()))))
}

#[derive(Default)]
struct MyApp {
    name: String,
    counter: i32,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading("My Application");
            ui.horizontal(|ui| {
                ui.label("Name:");
                ui.text_edit_singleline(&mut self.name);
            });
            ui.label(format!("Counter: {}", self.counter));
            ui.horizontal(|ui| {
                if ui.button("Increment").clicked() { self.counter += 1; }
                if ui.button("Decrement").clicked() { self.counter -= 1; }
            });
        });
    }
}
```

### Widgets and Layouts

```rust
impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // Side panel
        egui::SidePanel::left("nav").show(ctx, |ui| {
            ui.heading("Navigation");
            if ui.selectable_label(self.tab == 0, "Home").clicked() { self.tab = 0; }
            if ui.selectable_label(self.tab == 1, "Settings").clicked() { self.tab = 1; }
        });

        // Top panel
        egui::TopBottomPanel::top("toolbar").show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label("Status: Running");
                ui.separator();
                if ui.button("Refresh").clicked() { /* ... */ }
            });
        });

        // Central panel (fills remaining space)
        egui::CentralPanel::default().show(ctx, |ui| {
            // Common widgets
            ui.checkbox(&mut self.enabled, "Enable feature");
            ui.add(egui::Slider::new(&mut self.value, 0.0..=100.0).text("Volume"));
            ui.add(egui::DragValue::new(&mut self.value).speed(0.1));

            egui::ComboBox::from_label("Select")
                .selected_text(&self.options[self.selected])
                .show_ui(ui, |ui| {
                    for (i, opt) in self.options.iter().enumerate() {
                        ui.selectable_value(&mut self.selected, i, opt);
                    }
                });

            if ui.button("Open").clicked() {
                self.show_dialog = true;
            }

            // Modal window
            if self.show_dialog {
                egui::Window::new("Confirm")
                    .collapsible(false)
                    .show(ctx, |ui| {
                        ui.label("Are you sure?");
                        ui.horizontal(|ui| {
                            if ui.button("Yes").clicked() { self.show_dialog = false; }
                            if ui.button("No").clicked() { self.show_dialog = false; }
                        });
                    });
            }
        });
    }
}
```

### Async with egui

```rust
struct MyApp {
    runtime: tokio::runtime::Runtime,
    result: Arc<Mutex<Option<String>>>,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            if ui.button("Fetch").clicked() {
                let result = self.result.clone();
                let ctx = ctx.clone();
                self.runtime.spawn(async move {
                    let data = reqwest::get("https://api.example.com/data")
                        .await.unwrap().text().await.unwrap();
                    *result.lock().unwrap() = Some(data);
                    ctx.request_repaint();
                });
            }
            if let Some(data) = self.result.lock().unwrap().as_ref() {
                ui.label(data);
            }
        });
    }
}
```

## Desktop GUI — iced (Elm Architecture)

iced follows Model/Update/View with immutable messages. State changes only in `update`.

### Basic Application

```rust
// Cargo.toml: iced = { version = "0.12", features = ["tokio"] }
use iced::widget::{button, column, text};
use iced::{Element, Sandbox, Settings};

fn main() -> iced::Result {
    Counter::run(Settings::default())
}

#[derive(Default)]
struct Counter { value: i32 }

#[derive(Debug, Clone, Copy)]
enum Message { Increment, Decrement }

impl Sandbox for Counter {
    type Message = Message;

    fn new() -> Self { Self::default() }
    fn title(&self) -> String { String::from("Counter - iced") }

    fn update(&mut self, message: Message) {
        match message {
            Message::Increment => self.value += 1,
            Message::Decrement => self.value -= 1,
        }
    }

    fn view(&self) -> Element<Message> {
        column![
            button("Increment").on_press(Message::Increment),
            text(self.value).size(50),
            button("Decrement").on_press(Message::Decrement),
        ]
        .padding(20)
        .align_items(iced::Alignment::Center)
        .into()
    }
}
```

### Widgets and Forms

```rust
use iced::widget::{
    button, checkbox, column, container, horizontal_rule,
    pick_list, progress_bar, row, slider, text, text_input, toggler,
};
use iced::{Element, Length, Sandbox};

#[derive(Default)]
struct MyApp {
    username: String,
    password: String,
    remember_me: bool,
    volume: f32,
    selected_option: Option<String>,
}

#[derive(Debug, Clone)]
enum Message {
    UsernameChanged(String),
    PasswordChanged(String),
    RememberMeToggled(bool),
    VolumeChanged(f32),
    OptionSelected(String),
    Submit,
}

impl Sandbox for MyApp {
    type Message = Message;
    fn new() -> Self { Self { volume: 50.0, ..Default::default() } }
    fn title(&self) -> String { String::from("Form Demo") }

    fn update(&mut self, message: Message) {
        match message {
            Message::UsernameChanged(v) => self.username = v,
            Message::PasswordChanged(v) => self.password = v,
            Message::RememberMeToggled(v) => self.remember_me = v,
            Message::VolumeChanged(v) => self.volume = v,
            Message::OptionSelected(v) => self.selected_option = Some(v),
            Message::Submit => println!("Submitted: {}", self.username),
        }
    }

    fn view(&self) -> Element<Message> {
        let options: Vec<String> = vec!["Option 1", "Option 2", "Option 3"]
            .into_iter().map(String::from).collect();

        let content = column![
            text_input("Enter username", &self.username)
                .on_input(Message::UsernameChanged).padding(10),
            text_input("Enter password", &self.password)
                .on_input(Message::PasswordChanged).password().padding(10),
            horizontal_rule(10),
            checkbox("Remember me", self.remember_me)
                .on_toggle(Message::RememberMeToggled),
            text(format!("Volume: {:.0}", self.volume)),
            slider(0.0..=100.0, self.volume, Message::VolumeChanged),
            pick_list(options, self.selected_option.clone(), Message::OptionSelected)
                .placeholder("Select an option"),
            horizontal_rule(10),
            button("Submit").on_press(Message::Submit),
        ]
        .padding(20).spacing(10).max_width(400);

        container(content).width(Length::Fill).height(Length::Fill)
            .center_x().center_y().into()
    }
}
```

### Async Operations (Application Trait)

```rust
use iced::{Application, Command, Element, Settings, Theme};

struct AsyncApp {
    data: Option<String>,
    loading: bool,
}

#[derive(Debug, Clone)]
enum Message {
    FetchData,
    DataReceived(Result<String, String>),
}

impl Application for AsyncApp {
    type Message = Message;
    type Theme = Theme;
    type Executor = iced::executor::Default;
    type Flags = ();

    fn new(_flags: ()) -> (Self, Command<Message>) {
        (Self { data: None, loading: false }, Command::none())
    }

    fn title(&self) -> String { String::from("Async App") }

    fn update(&mut self, message: Message) -> Command<Message> {
        match message {
            Message::FetchData => {
                self.loading = true;
                Command::perform(
                    async { reqwest::get("https://api.example.com")
                        .await.map(|r| r.text()).map_err(|e| e.to_string()) },
                    Message::DataReceived,
                )
            }
            Message::DataReceived(result) => {
                self.loading = false;
                match result {
                    Ok(data) => self.data = Some(data),
                    Err(_) => {}
                }
                Command::none()
            }
        }
    }

    fn view(&self) -> Element<Message> { /* ... */ }
}
```

### egui vs iced Comparison

| Aspect | egui | iced |
|--------|------|------|
| **Architecture** | Immediate mode | Elm (Model/Update/View) |
| **State** | Direct mutation | Message-based |
| **Learning curve** | Lower | Higher |
| **Styling** | Limited, procedural | Theme-based, customizable |
| **Async** | Manual (tokio runtime) | Built-in Command system |
| **Best for** | Tools, debug UIs, prototyping | Production apps, complex UIs |

## Web Frontend — Leptos

Full-stack Rust web framework with fine-grained reactivity and first-class SSR.

### Reactive Signals

```rust
// Cargo.toml: leptos = "0.6"
use leptos::*;

#[component]
fn Counter() -> impl IntoView {
    let (count, set_count) = signal(0);
    let doubled = move || count.get() * 2;  // Derived signal

    view! {
        <div>
            <p>"Count: " {count}</p>
            <p>"Doubled: " {doubled}</p>
            <button on:click=move |_| set_count.update(|n| *n += 1)>
                "+1"
            </button>
        </div>
    }
}

fn main() {
    leptos::mount_to_body(Counter);
}
```

### Component Props

```rust
#[component]
fn Greeting(
    name: String,
    #[prop(default = "Hello".to_string())]
    greeting: String,
    #[prop(optional)]
    excited: bool,
) -> impl IntoView {
    let punct = if excited { "!" } else { "." };
    view! { <p>{greeting}", "{name}{punct}</p> }
}

#[component]
fn App() -> impl IntoView {
    view! {
        <Greeting name="World".to_string() />
        <Greeting name="Rust".to_string() greeting="Welcome".to_string() />
        <Greeting name="Leptos".to_string() excited=true />
    }
}
```

### Conditional Rendering and Lists

```rust
#[component]
fn TodoList() -> impl IntoView {
    let (todos, set_todos) = signal(vec![
        ("Buy groceries".to_string(), false),
        ("Write code".to_string(), true),
    ]);
    let (new_todo, set_new_todo) = signal(String::new());

    let add_todo = move |_| {
        let value = new_todo.get();
        if !value.is_empty() {
            set_todos.update(|t| t.push((value, false)));
            set_new_todo.set(String::new());
        }
    };

    view! {
        <input
            type="text" placeholder="New todo..."
            prop:value=new_todo
            on:input=move |ev| set_new_todo.set(event_target_value(&ev))
        />
        <button on:click=add_todo>"Add"</button>

        <Show when=move || logged_in.get()
              fallback=|| view! { <p>"Please log in"</p> }>
            <p>"Welcome back!"</p>
        </Show>

        <ul>
            <For
                each=move || todos.get().into_iter().enumerate()
                key=|(i, _)| *i
                children=move |(i, (text, completed))| {
                    view! { <li class:completed=completed><span>{text}</span></li> }
                }
            />
        </ul>
    }
}
```

### Server Functions and Routing

```rust
use leptos::*;
use leptos_router::*;

// Server function — runs on server, callable from client
#[server]
pub async fn get_users() -> Result<Vec<User>, ServerFnError> {
    sqlx::query_as!(User, "SELECT * FROM users")
        .fetch_all(&pool).await.map_err(Into::into)
}

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <nav>
                <A href="/">"Home"</A>
                <A href="/users">"Users"</A>
            </nav>
            <Routes>
                <Route path="/" view=HomePage />
                <Route path="/users" view=UserList />
                <Route path="/users/:id" view=UserDetail />
                <Route path="/*any" view=NotFound />
            </Routes>
        </Router>
    }
}

#[component]
fn UserList() -> impl IntoView {
    let users = Resource::new(|| (), |_| async { get_users().await });

    view! {
        <Suspense fallback=move || view! { <p>"Loading..."</p> }>
            {move || users.get().map(|result| match result {
                Ok(users) => view! {
                    <ul>
                        {users.into_iter().map(|u| view! {
                            <li>{u.name}" - "{u.email}</li>
                        }).collect_view()}
                    </ul>
                }.into_view(),
                Err(e) => view! { <p>"Error: "{e.to_string()}</p> }.into_view(),
            })}
        </Suspense>
    }
}

#[component]
fn UserDetail() -> impl IntoView {
    let params = use_params_map();
    let id = move || params.with(|p| p.get("id").cloned().unwrap_or_default());
    view! { <p>"User ID: "{id}</p> }
}
```

### SSR Setup with Axum

```rust
use axum::Router;
use leptos::*;
use leptos_axum::{generate_route_list, LeptosRoutes};

#[tokio::main]
async fn main() {
    let conf = get_configuration(None).await.unwrap();
    let leptos_options = conf.leptos_options;
    let routes = generate_route_list(App);

    let app = Router::new()
        .leptos_routes(&leptos_options, routes, App)
        .fallback(leptos_axum::file_and_error_handler(App));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

## Web Frontend — Yew

React-like component framework using virtual DOM.

### Function Components and State

```rust
// Cargo.toml: yew = { version = "0.21", features = ["csr"] }
use yew::prelude::*;

#[function_component(Counter)]
fn counter() -> Html {
    let count = use_state(|| 0);

    let increment = {
        let count = count.clone();
        Callback::from(move |_| count.set(*count + 1))
    };
    let decrement = {
        let count = count.clone();
        Callback::from(move |_| count.set(*count - 1))
    };

    html! {
        <div>
            <button onclick={decrement}>{ "-1" }</button>
            <span>{ *count }</span>
            <button onclick={increment}>{ "+1" }</button>
        </div>
    }
}

fn main() { yew::Renderer::<Counter>::new().render(); }
```

### Props and Forms

```rust
#[derive(Properties, PartialEq)]
pub struct GreetingProps {
    pub name: String,
    #[prop_or("Hello".to_string())]
    pub greeting: String,
    #[prop_or_default]
    pub excited: bool,
}

#[function_component(Greeting)]
fn greeting(props: &GreetingProps) -> Html {
    let punct = if props.excited { "!" } else { "." };
    html! { <p>{ format!("{}, {}{}", props.greeting, props.name, punct) }</p> }
}

#[function_component(FormExample)]
fn form_example() -> Html {
    let username = use_state(String::new);
    let on_input = {
        let username = username.clone();
        Callback::from(move |e: InputEvent| {
            let input: web_sys::HtmlInputElement = e.target_unchecked_into();
            username.set(input.value());
        })
    };

    html! {
        <form>
            <input type="text" value={(*username).clone()} oninput={on_input} />
            if !username.is_empty() {
                <p>{ format!("Hello, {}!", *username) }</p>
            }
        </form>
    }
}
```

### Data Fetching

```rust
use gloo_net::http::Request;
use serde::Deserialize;

#[derive(Clone, PartialEq, Deserialize)]
struct User { id: u32, name: String, email: String }

#[function_component(UserList)]
fn user_list() -> Html {
    let users = use_state(|| None::<Vec<User>>);
    let loading = use_state(|| true);

    {
        let users = users.clone();
        let loading = loading.clone();
        use_effect_with((), move |_| {
            wasm_bindgen_futures::spawn_local(async move {
                match Request::get("/api/users").send().await {
                    Ok(resp) => {
                        if let Ok(data) = resp.json::<Vec<User>>().await {
                            users.set(Some(data));
                        }
                    }
                    Err(_) => {}
                }
                loading.set(false);
            });
            || ()
        });
    }

    if *loading { return html! { <p>{"Loading..."}</p> }; }

    match (*users).clone() {
        Some(users) => html! {
            <ul>{ for users.iter().map(|u| html! {
                <li key={u.id}>{ format!("{} - {}", u.name, u.email) }</li>
            }) }</ul>
        },
        None => html! { <p>{"No users found"}</p> },
    }
}
```

### Context (Global State) and Routing

```rust
use std::rc::Rc;

#[derive(Clone, PartialEq)]
struct AppState { theme: String, user: Option<String> }

impl Reducible for AppState {
    type Action = AppAction;
    fn reduce(self: Rc<Self>, action: Self::Action) -> Rc<Self> {
        match action {
            AppAction::SetTheme(t) => Rc::new(AppState { theme: t, ..(*self).clone() }),
            AppAction::Login(u) => Rc::new(AppState { user: Some(u), ..(*self).clone() }),
            AppAction::Logout => Rc::new(AppState { user: None, ..(*self).clone() }),
        }
    }
}

type AppContext = UseReducerHandle<AppState>;

#[function_component(App)]
fn app() -> Html {
    let state = use_reducer(|| AppState { theme: "light".into(), user: None });
    html! {
        <ContextProvider<AppContext> context={state}>
            <Header />
        </ContextProvider<AppContext>>
    }
}

// Routing
use yew_router::prelude::*;

#[derive(Clone, Routable, PartialEq)]
enum Route {
    #[at("/")] Home,
    #[at("/users/:id")] UserDetail { id: u32 },
    #[not_found] #[at("/404")] NotFound,
}

fn switch(route: Route) -> Html {
    match route {
        Route::Home => html! { <h1>{"Home"}</h1> },
        Route::UserDetail { id } => html! { <p>{format!("User {id}")}</p> },
        Route::NotFound => html! { <h1>{"404"}</h1> },
    }
}

#[function_component(App)]
fn routed_app() -> Html {
    html! {
        <BrowserRouter>
            <nav>
                <Link<Route> to={Route::Home}>{"Home"}</Link<Route>>
            </nav>
            <Switch<Route> render={switch} />
        </BrowserRouter>
    }
}
```

## Web Frontend — Dioxus

Multi-platform framework (web, desktop, mobile) with React-like signals and RSX syntax.

### Basic Component

```rust
// Cargo.toml: dioxus = { version = "0.6", features = ["web"] }
use dioxus::prelude::*;

fn main() {
    dioxus::launch(App);
}

#[component]
fn App() -> Element {
    let mut count = use_signal(|| 0);

    rsx! {
        div {
            h1 { "Counter: {count}" }
            button { onclick: move |_| count += 1, "+1" }
            button { onclick: move |_| count -= 1, "-1" }
        }
    }
}
```

### Props and Context

```rust
#[component]
fn UserCard(name: String, #[props(default = false)] admin: bool) -> Element {
    rsx! {
        div { class: if admin { "user admin" } else { "user" },
            p { "{name}" }
            if admin { span { "⭐ Admin" } }
        }
    }
}

// Global state via context
#[component]
fn App() -> Element {
    use_context_provider(|| Signal::new(AppState::default()));

    rsx! {
        Router::<Route> {}
    }
}

// Routing
#[derive(Clone, Routable, PartialEq)]
enum Route {
    #[route("/")]
    Home {},
    #[route("/users/:id")]
    UserDetail { id: u32 },
}
```

### Server Functions

```rust
#[server]
async fn get_users() -> Result<Vec<User>, ServerFnError> {
    // Runs on server, callable from client
    sqlx::query_as!(User, "SELECT * FROM users")
        .fetch_all(&pool).await.map_err(Into::into)
}

#[component]
fn UserList() -> Element {
    let users = use_server_future(get_users)?;

    rsx! {
        match &*users.read() {
            Some(Ok(users)) => rsx! {
                for user in users {
                    p { "{user.name}" }
                }
            },
            Some(Err(e)) => rsx! { p { "Error: {e}" } },
            None => rsx! { p { "Loading..." } },
        }
    }
}
```

### Framework Comparison

| Aspect | Leptos | Yew | Dioxus |
|--------|--------|-----|--------|
| **Reactivity** | Fine-grained signals | Virtual DOM | Signals |
| **SSR** | First-class | Requires setup | Built-in |
| **Bundle size** | Smallest | Largest | Small |
| **Server functions** | `#[server]` | Manual API | `#[server]` |
| **Syntax** | `view!` macro | `html!` macro | `rsx!` macro |
| **Platforms** | Web only | Web only | Web, desktop, mobile |
| **Hot reload** | CSS only | No | RSX hot reload |
| **Ecosystem** | Growing fast | Most mature | Growing |
| **Best for** | Full-stack web | SPA with React exp. | Multi-platform |

### Leptos Build Mode Requirement

Leptos requires exactly one render mode feature per build target:

```toml
# Cargo.toml — pick exactly ONE per target
[features]
csr = ["leptos/csr"]      # Client-side rendering only
hydrate = ["leptos/hydrate"]  # Hydrate server-rendered HTML
ssr = ["leptos/ssr"]      # Server-side rendering
```

Enabling zero or multiple modes causes silent failures or wrong output.

## Build Tools

### Trunk (Recommended)

```bash
cargo install trunk
trunk serve          # Dev server with hot reload
trunk build --release  # Production build
```

```toml
# Trunk.toml
[build]
target = "index.html"
dist = "dist"
```

### wasm-bindgen CLI (for libraries)

```bash
cargo install wasm-bindgen-cli
cargo build --target wasm32-unknown-unknown --release
wasm-bindgen target/wasm32-unknown-unknown/release/my_lib.wasm \
    --out-dir pkg --target web
```

**Note:** wasm-pack was archived in January 2025. Use Trunk for web apps or wasm-bindgen CLI for libraries.

## WebAssembly — Browser Integration

### Project Setup

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"
web-sys = { version = "0.3", features = ["console", "Document", "Window", "Element"] }

[dev-dependencies]
wasm-bindgen-test = "0.3"

[profile.release]
opt-level = "s"    # Optimize for size
lto = true
```

### Exposing Rust to JavaScript

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[wasm_bindgen]
pub struct Point { x: f64, y: f64 }

#[wasm_bindgen]
impl Point {
    #[wasm_bindgen(constructor)]
    pub fn new(x: f64, y: f64) -> Point { Point { x, y } }

    #[wasm_bindgen(getter)]
    pub fn x(&self) -> f64 { self.x }

    #[wasm_bindgen(setter)]
    pub fn set_x(&mut self, x: f64) { self.x = x; }

    pub fn distance_to(&self, other: &Point) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }

    pub fn origin() -> Point { Point { x: 0.0, y: 0.0 } }
}
```

### Complex Types and JS Objects

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Array, Object, Reflect};

// Typed arrays pass efficiently (zero-copy view into WASM memory)
#[wasm_bindgen]
pub fn sum_array(arr: &[i32]) -> i32 { arr.iter().sum() }

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> Vec<u32> {
    let mut fib = vec![0, 1];
    for i in 2..n as usize { fib.push(fib[i - 1] + fib[i - 2]); }
    fib
}

// JavaScript object interop
#[wasm_bindgen]
pub fn process_config(config: JsValue) -> Result<String, JsValue> {
    let obj: Object = config.dyn_into()?;
    let name = Reflect::get(&obj, &"name".into())?
        .as_string().ok_or("name must be a string")?;
    let count = Reflect::get(&obj, &"count".into())?
        .as_f64().ok_or("count must be a number")? as u32;
    Ok(format!("{} x {}", name, count))
}
```

### Calling JavaScript from Rust

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    fn alert(s: &str);

    #[wasm_bindgen(js_namespace = myApp)]
    fn sendAnalytics(event: &str, data: JsValue);
}

macro_rules! console_log {
    ($($t:tt)*) => (log(&format!($($t)*)))
}
```

### DOM Manipulation

```rust
use web_sys::{Document, HtmlElement};

#[wasm_bindgen]
pub fn manipulate_dom() -> Result<(), JsValue> {
    let window = web_sys::window().expect("no global window");
    let document = window.document().expect("no document");

    let div = document.create_element("div")?;
    div.set_id("rust-created");
    div.set_inner_html("<p>Created by Rust!</p>");

    document.body().expect("no body").append_child(&div)?;

    // Event listener
    let closure = Closure::wrap(Box::new(|_event: web_sys::MouseEvent| {
        web_sys::console::log_1(&"Clicked!".into());
    }) as Box<dyn FnMut(_)>);

    div.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();  // Don't drop — event listener needs it alive

    Ok(())
}
```

### Async JavaScript Interop

```rust
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response};

#[wasm_bindgen]
pub async fn fetch_data(url: &str) -> Result<JsValue, JsValue> {
    let window = web_sys::window().unwrap();

    let mut opts = RequestInit::new();
    opts.method("GET");
    let request = Request::new_with_str_and_init(url, &opts)?;

    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;
    let json = JsFuture::from(resp.json()?).await?;
    Ok(json)
}
```

### Canvas Rendering

```rust
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

#[wasm_bindgen]
pub struct Canvas {
    ctx: CanvasRenderingContext2d,
    width: u32,
    height: u32,
}

#[wasm_bindgen]
impl Canvas {
    #[wasm_bindgen(constructor)]
    pub fn new(canvas_id: &str) -> Result<Canvas, JsValue> {
        let document = web_sys::window().unwrap().document().unwrap();
        let canvas = document.get_element_by_id(canvas_id).unwrap()
            .dyn_into::<HtmlCanvasElement>()?;
        let ctx = canvas.get_context("2d")?.unwrap()
            .dyn_into::<CanvasRenderingContext2d>()?;
        Ok(Canvas { width: canvas.width(), height: canvas.height(), ctx })
    }

    pub fn clear(&self) {
        self.ctx.clear_rect(0.0, 0.0, self.width as f64, self.height as f64);
    }

    pub fn draw_rect(&self, x: f64, y: f64, w: f64, h: f64, color: &str) {
        self.ctx.set_fill_style(&color.into());
        self.ctx.fill_rect(x, y, w, h);
    }

    pub fn draw_circle(&self, x: f64, y: f64, radius: f64, color: &str) {
        self.ctx.begin_path();
        self.ctx.arc(x, y, radius, 0.0, std::f64::consts::PI * 2.0).unwrap();
        self.ctx.set_fill_style(&color.into());
        self.ctx.fill();
    }
}
```

### HTML Integration

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Rust WASM</title></head>
<body>
    <div id="app"></div>
    <script type="module">
        import init, { greet, Point } from './pkg/my_lib.js';
        async function main() {
            await init();
            console.log(greet("World"));
            const p1 = new Point(0, 0);
            const p2 = new Point(3, 4);
            console.log("Distance:", p1.distance_to(p2));
        }
        main();
    </script>
</body>
</html>
```

## WebAssembly — Server-Side

### Wasmtime Runtime

```rust
// Cargo.toml: wasmtime = "18"
use wasmtime::*;

fn main() -> Result<()> {
    let engine = Engine::default();
    let mut store = Store::new(&engine, ());

    let module = Module::from_file(&engine, "module.wasm")?;
    let instance = Instance::new(&mut store, &module, &[])?;

    let add = instance.get_typed_func::<(i32, i32), i32>(&mut store, "add")?;
    let result = add.call(&mut store, (2, 3))?;
    println!("2 + 3 = {}", result);

    Ok(())
}
```

### Host Functions

```rust
fn main() -> Result<()> {
    let engine = Engine::default();
    let mut store = Store::new(&engine, HostState { log_count: 0 });

    let log_func = Func::wrap(&mut store, |mut caller: Caller<'_, HostState>, ptr: i32, len: i32| {
        let memory = caller.get_export("memory")
            .and_then(|e| e.into_memory()).expect("missing memory");
        let data = memory.data(&caller);
        let message = std::str::from_utf8(&data[ptr as usize..(ptr + len) as usize]).unwrap();
        println!("[WASM LOG]: {}", message);
        caller.data_mut().log_count += 1;
    });

    let mut linker = Linker::new(&engine);
    linker.define(&store, "env", "log", log_func)?;

    let module = Module::from_file(&engine, "module.wasm")?;
    let instance = linker.instantiate(&mut store, &module)?;

    let run = instance.get_typed_func::<(), ()>(&mut store, "run")?;
    run.call(&mut store)?;

    Ok(())
}

struct HostState { log_count: u32 }
```

### Wasmer Runtime

```rust
// Cargo.toml: wasmer = "4"
use wasmer::{imports, Instance, Module, Store, Value};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut store = Store::default();
    let module = Module::from_file(&store, "module.wasm")?;
    let instance = Instance::new(&mut store, &module, &imports! {})?;

    let add = instance.exports.get_function("add")?;
    let result = add.call(&mut store, &[Value::I32(2), Value::I32(3)])?;
    println!("Result: {:?}", result);

    Ok(())
}
```

### WASI (WebAssembly System Interface)

```rust
// Standard Rust code works with WASI target
use std::{env, fs};

fn main() {
    if let Ok(value) = env::var("MY_VAR") {
        println!("MY_VAR = {}", value);
    }

    let content = fs::read_to_string("/input/data.txt").expect("Failed to read");
    fs::write("/output/result.txt", content.to_uppercase()).expect("Failed to write");
}
```

```bash
rustup target add wasm32-wasi
cargo build --target wasm32-wasi --release

# Run with Wasmtime (grant directory access)
wasmtime --dir=/input --dir=/output target/wasm32-wasi/release/my_app.wasm
```

## WASM Testing and Optimization

### Browser Tests

```rust
// tests/web.rs
#![cfg(target_arch = "wasm32")]
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_add() { assert_eq!(my_lib::add(2, 3), 5); }

#[wasm_bindgen_test]
async fn test_async_op() {
    let result = my_lib::async_operation().await;
    assert!(result.is_ok());
}
```

```bash
# Run browser tests with wasm-bindgen-test-runner (wasm-pack is archived)
cargo test --target wasm32-unknown-unknown
# Or with specific browser:
WASM_BINDGEN_TEST_TIMEOUT=60 cargo test --target wasm32-unknown-unknown
```

### Size Optimization

```toml
[profile.release]
opt-level = "z"      # Smallest binary
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

```bash
# Further optimization with wasm-opt (Binaryen)
wasm-opt -Os -o optimized.wasm pkg/my_lib_bg.wasm

# Inspect with wasm2wat
wasm2wat module.wasm > module.wat
```

### Performance Tips

```rust
// BAD: Many small JS<->WASM boundary crossings
#[wasm_bindgen]
pub fn process_one(x: f64) -> f64 { x * 2.0 }
// Called 1000 times = 1000 crossings

// GOOD: Batch operations — one crossing
#[wasm_bindgen]
pub fn process_batch(data: &[f64]) -> Vec<f64> {
    data.iter().map(|x| x * 2.0).collect()
}

// GOOD: Pre-allocate and reuse buffers
#[wasm_bindgen]
pub struct ImageProcessor { buffer: Vec<u8> }

#[wasm_bindgen]
impl ImageProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(size: usize) -> Self { Self { buffer: vec![0; size] } }

    pub fn process(&mut self, input: &[u8]) -> &[u8] {
        self.buffer.copy_from_slice(input);
        // ... process ...
        &self.buffer
    }
}
```

### no_std for Minimal Size

```rust
#![no_std]
extern crate alloc;
use alloc::vec::Vec;
use wasm_bindgen::prelude::*;

use core::panic::PanicInfo;
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! { loop {} }

#[wasm_bindgen]
pub fn process(data: &[u8]) -> Vec<u8> {
    data.iter().map(|b| b.wrapping_add(1)).collect()
}
```

## WASM Memory Model

WebAssembly uses a single contiguous block of linear memory:

```
Traditional RAM:          WASM Linear Memory:
┌───┐ ┌───┐ ┌───┐        ┌───┬───┬───┬───┬───┬───┐
│ A │ │ B │ │ C │   vs   │ 0 │ 1 │ 2 │ 3 │...│ N │
└───┘ └───┘ └───┘        └───┴───┴───┴───┴───┴───┘
(scattered)               (contiguous, indexed)
```

**Benefits:** Cache-friendly sequential access, simpler security model (bounded), predictable performance.

**Security:** WASM modules are sandboxed from each other and the host. Rust's ownership prevents most buffer overflows within the module.

### C ABI Memory Management (Advanced)

For maximum control without wasm-bindgen overhead:

```rust
#![no_std]
extern crate alloc;
use alloc::alloc::{alloc, dealloc, Layout};

#[no_mangle]
pub unsafe extern "C" fn ns_malloc(size: u32, alignment: u32) -> *mut u8 {
    let layout = Layout::from_size_align_unchecked(size as usize, alignment as usize);
    alloc(layout)
}

#[no_mangle]
pub unsafe extern "C" fn ns_free(ptr: *mut u8, size: u32, alignment: u32) {
    let layout = Layout::from_size_align_unchecked(size as usize, alignment as usize);
    dealloc(ptr, layout);
}
```

```javascript
const ptr = instance.exports.ns_malloc(1024, 8);
const view = new Uint8Array(instance.exports.memory.buffer, ptr, 1024);
view.set(myData);
instance.exports.process_buffer(ptr, 1024);
instance.exports.ns_free(ptr, 1024, 8);
```

## When to Use What

| Use Case | Recommendation |
|----------|---------------|
| Internal tools, debug UIs | **egui** — fastest to build |
| Production desktop app | **iced** — structured state management |
| Full-stack web app (SSR) | **Leptos** — signals, server functions |
| SPA with React experience | **Yew** — familiar virtual DOM |
| Performance-critical browser code | **wasm-bindgen** — direct WASM |
| Sandboxed plugin system | **Wasmtime/Wasmer** — server-side WASM |
| Edge computing | **WASI** — portable serverless |
| Shared Rust logic in browser | **cdylib + wasm-bindgen** |

## Related Skills

- **[rust-wasm](../../rust-wasm/SKILL.md)** — Detailed Leptos/Yew/Dioxus patterns, Phoenix LiveView + WASM integration, Tailwind CSS with WASM apps
