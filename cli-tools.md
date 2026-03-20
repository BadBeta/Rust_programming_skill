# CLI Tools in Rust

Building robust, user-friendly command-line interfaces with argument parsing (clap derive and builder), user input handling, styled output (colored), progress indicators (indicatif), tables (prettytable), terminal manipulation (crossterm), file system operations, signal handling, and error handling with anyhow.

## Rules for CLI Tools (LLM)

1. **ALWAYS return `ExitCode` from `main()` instead of calling `std::process::exit()`** — `exit()` skips destructors and cleanup; `ExitCode` enables graceful shutdown (pattern from ripgrep)
2. **ALWAYS handle broken pipe errors gracefully** — when piped to `head`/`grep`, writes fail with `BrokenPipe`; detect this and exit with code 0 (Unix convention)
3. **ALWAYS use `std::io::IsTerminal` to detect interactive vs piped output** — disable colors, progress bars, and interactive prompts when piped; `atty` crate is deprecated since Rust 1.70
4. **ALWAYS write errors to stderr, data to stdout** — use `eprintln!` for errors; this allows `myapp 2>/dev/null` to suppress errors while keeping data
5. **NEVER use `unwrap()` in CLI applications** — use `anyhow::Result` with `.context()` for user-friendly error messages; panics produce confusing output for end users
6. **ALWAYS validate all arguments before executing** — parse and validate the full CLI first, then execute; don't partially execute before discovering invalid flags
7. **PREFER `clap` derive over builder API** — derive is more concise, type-safe, and self-documenting; use builder only for dynamic flag generation
8. **ALWAYS follow config precedence: CLI args > env vars > config file > defaults** — clap supports this with `#[arg(env = "VAR")]` and `#[serde(default)]`

### Common Mistakes (BAD/GOOD)

**Using `process::exit` instead of `ExitCode`:**
```rust
// BAD: skips destructors, buffers may not flush, files may not close
fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {e}");
        std::process::exit(1);  // destructors skipped!
    }
}

// GOOD: ExitCode allows graceful cleanup (pattern from ripgrep)
fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("Error: {e:#}");
            ExitCode::from(2)
        }
    }
}
```

**Ignoring broken pipe errors:**
```rust
// BAD: prints ugly error when piped to head/grep
// "Error: Broken pipe (os error 32)"

// GOOD: detect broken pipe and exit gracefully (ripgrep pattern)
fn main() -> ExitCode {
    match run() {
        Ok(code) => code,
        Err(err) => {
            // Walk the error chain looking for BrokenPipe
            if err.chain().any(|cause| {
                cause.downcast_ref::<std::io::Error>()
                    .is_some_and(|e| e.kind() == std::io::ErrorKind::BrokenPipe)
            }) {
                return ExitCode::SUCCESS;  // expected when piped to head
            }
            eprintln!("Error: {err:#}");
            ExitCode::from(2)
        }
    }
}
```

**Not gating interactive features on terminal detection:**
```rust
// BAD: progress bar breaks piped output
let pb = ProgressBar::new(total);  // always shows progress

// GOOD: only show progress in interactive terminals
use std::io::IsTerminal;
let pb = if std::io::stderr().is_terminal() {
    ProgressBar::new(total)
} else {
    ProgressBar::hidden()
};
```

## Argument Parsing with clap

### Basic CLI with Derive Macros

```rust
use clap::Parser;

/// A simple CLI application that greets a user.
#[derive(Parser, Debug)]
#[command(author, version, about, long_about = None)]
struct Args {
    /// The name of the person to greet
    #[arg(short, long)]
    name: String,

    /// Number of times to repeat the greeting
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}

fn main() {
    let args = Args::parse();

    for _ in 0..args.count {
        println!("Hello, {}!", args.name);
    }
}
```

Usage:
```bash
$ myapp --name Alice --count 3
Hello, Alice!
Hello, Alice!
Hello, Alice!

$ myapp --help
A simple CLI application that greets a user.

Usage: myapp [OPTIONS] --name <NAME>

Options:
  -n, --name <NAME>    The name of the person to greet
  -c, --count <COUNT>  Number of times to repeat [default: 1]
  -h, --help           Print help
  -V, --version        Print version
```

### Positional Arguments

```rust
use clap::Parser;
use std::path::PathBuf;

/// A CLI application that processes files.
#[derive(Parser, Debug)]
#[command(author, version, about)]
struct Args {
    /// The path to the file to process (positional)
    #[arg(value_name = "FILE")]
    file: PathBuf,

    /// Optional output directory
    #[arg(short, long)]
    output: Option<PathBuf>,

    /// Enable verbose output
    #[arg(short, long, action)]
    verbose: bool,
}

fn main() {
    let args = Args::parse();

    println!("Processing file: {:?}", args.file);
    if let Some(output_dir) = args.output {
        println!("Output directory: {:?}", output_dir);
    }
    if args.verbose {
        println!("Verbose mode enabled");
    }
}
```

Usage:
```bash
$ myapp input.txt                    # Positional argument
$ myapp input.txt -o /tmp/output     # With optional flag
$ myapp -v input.txt                 # With verbose flag
```

### Subcommands

```rust
use clap::{Parser, Subcommand};
use std::path::PathBuf;

/// A CLI application for managing resources.
#[derive(Parser, Debug)]
#[command(
    author, version, about, long_about = None,
    arg_required_else_help = true,    // Show help if no args/subcommand given
    propagate_version = true,         // Subcommands inherit parent version
)]
struct Cli {
    /// Optional global configuration file
    #[arg(short, long, global = true)]
    config: Option<PathBuf>,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Creates a new resource
    Create {
        /// The name of the resource to create
        #[arg(short, long)]
        name: String,

        /// Optional type of the resource
        #[arg(long, default_value = "default")]
        resource_type: String,
    },

    /// Deletes an existing resource
    Delete {
        /// The ID of the resource to delete (positional)
        id: u32,

        /// Force deletion without confirmation
        #[arg(short, long, action)]
        force: bool,
    },

    /// Lists all resources
    List {
        /// Filter by status
        #[arg(short, long)]
        status: Option<String>,

        /// Output format
        #[arg(long, value_enum, default_value = "table")]
        format: OutputFormat,
    },
}

#[derive(Debug, Clone, clap::ValueEnum)]
enum OutputFormat {
    Table,
    Json,
    Csv,
}

fn main() {
    let cli = Cli::parse();

    // Access global arguments
    if let Some(config_path) = cli.config {
        println!("Using configuration file: {:?}", config_path);
    }

    // Match on subcommands
    match cli.command {
        Commands::Create { name, resource_type } => {
            println!("Creating resource '{}' of type '{}'", name, resource_type);
        }
        Commands::Delete { id, force } => {
            if force {
                println!("Force deleting resource {}", id);
            } else {
                println!("Deleting resource {}", id);
            }
        }
        Commands::List { status, format } => {
            println!("Listing resources (status: {:?}, format: {:?})", status, format);
        }
    }
}
```

Usage:
```bash
$ myapp --config settings.toml create --name my-resource
$ myapp create --name another --resource-type database
$ myapp delete 123 --force
$ myapp list --status active --format json
$ myapp create --help   # Subcommand-specific help
```

### Nested Subcommands

```rust
use clap::{Parser, Subcommand, Args};

#[derive(Parser)]
#[command(author, version, about)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// User management commands
    User(UserArgs),
    /// Database operations
    Db(DbArgs),
}

#[derive(Args)]
struct UserArgs {
    #[command(subcommand)]
    command: UserCommands,
}

#[derive(Subcommand)]
enum UserCommands {
    /// Create a new user
    Create { username: String },
    /// Delete a user
    Delete { user_id: u32 },
    /// List all users
    List,
}

#[derive(Args)]
struct DbArgs {
    #[command(subcommand)]
    command: DbCommands,
}

#[derive(Subcommand)]
enum DbCommands {
    /// Run migrations
    Migrate,
    /// Seed the database
    Seed {
        #[arg(long)]
        count: u32,
    },
    /// Reset the database
    Reset {
        #[arg(long, action)]
        confirm: bool,
    },
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::User(user) => match user.command {
            UserCommands::Create { username } => println!("Creating user: {}", username),
            UserCommands::Delete { user_id } => println!("Deleting user: {}", user_id),
            UserCommands::List => println!("Listing users"),
        },
        Commands::Db(db) => match db.command {
            DbCommands::Migrate => println!("Running migrations"),
            DbCommands::Seed { count } => println!("Seeding {} records", count),
            DbCommands::Reset { confirm } => {
                if confirm {
                    println!("Resetting database");
                } else {
                    println!("Use --confirm to reset");
                }
            }
        },
    }
}
```

Usage:
```bash
$ myapp user create alice
$ myapp user delete 123
$ myapp db migrate
$ myapp db seed --count 100
$ myapp db reset --confirm
```

### Value Validation

```rust
use clap::Parser;

#[derive(Parser)]
struct Args {
    /// Port number (1-65535)
    #[arg(short, long, value_parser = clap::value_parser!(u16).range(1..=65535))]
    port: u16,

    /// Log level
    #[arg(long, value_parser = ["debug", "info", "warn", "error"])]
    log_level: String,

    /// Number of workers (1-100)
    #[arg(long, default_value_t = 4, value_parser = clap::value_parser!(u32).range(1..=100))]
    workers: u32,
}
```

### Environment Variables

```rust
use clap::Parser;

#[derive(Parser)]
struct Args {
    /// Database URL (can also use DATABASE_URL env var)
    #[arg(long, env = "DATABASE_URL")]
    database_url: String,

    /// API key (can also use API_KEY env var)
    #[arg(long, env = "API_KEY")]
    api_key: Option<String>,
}
```

### Clap + Serde JSON Config Integration

Pass complex configuration as JSON arguments:

```rust
use clap::Parser;
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct TaskConfig {
    timeout_ms: u64,
    retries: u32,
    endpoints: Vec<String>,
}

#[derive(Parser, Debug)]
#[command(author, version, about)]
struct Args {
    /// JSON configuration string
    #[arg(short, long)]
    config_json: String,

    /// Or load config from file
    #[arg(long)]
    config_file: Option<std::path::PathBuf>,
}

fn main() -> anyhow::Result<()> {
    use anyhow::Context;

    let args = Args::parse();

    let config: TaskConfig = if let Some(path) = args.config_file {
        let content = std::fs::read_to_string(&path)
            .with_context(|| format!("Failed to read config: {}", path.display()))?;
        serde_json::from_str(&content)?
    } else {
        serde_json::from_str(&args.config_json)
            .context("Failed to parse --config-json")?
    };

    println!("Config: {:?}", config);
    Ok(())
}

// Usage:
// myapp --config-json '{"timeout_ms": 5000, "retries": 3, "endpoints": ["http://a", "http://b"]}'
// myapp --config-file config.json
```

## User Input and Output

### Reading User Input

```rust
use std::io::{self, Write};

fn main() -> io::Result<()> {
    // Simple prompt
    print!("Please enter your name: ");
    io::stdout().flush()?;  // Ensure prompt is displayed immediately

    let mut name = String::new();
    io::stdin().read_line(&mut name)?;

    let trimmed_name = name.trim();  // Remove trailing newline
    println!("Hello, {}!", trimmed_name);

    Ok(())
}
```

### Input with Validation Loop

```rust
use std::io::{self, Write};

fn read_number(prompt: &str) -> io::Result<u32> {
    let mut input = String::new();

    loop {
        print!("{}", prompt);
        io::stdout().flush()?;

        input.clear();
        io::stdin().read_line(&mut input)?;

        match input.trim().parse::<u32>() {
            Ok(num) => return Ok(num),
            Err(_) => {
                eprintln!("Invalid input. Please enter a valid number.");
            }
        }
    }
}

fn main() -> io::Result<()> {
    let age = read_number("Please enter your age: ")?;
    println!("You are {} years old.", age);
    Ok(())
}
```

### Confirmation Prompts

```rust
use std::io::{self, Write};

fn ask_confirmation(prompt: &str) -> io::Result<bool> {
    let mut input = String::new();

    loop {
        print!("{} (y/N): ", prompt);
        io::stdout().flush()?;

        input.clear();
        io::stdin().read_line(&mut input)?;

        match input.trim().to_lowercase().as_str() {
            "y" | "yes" => return Ok(true),
            "n" | "no" | "" => return Ok(false),  // Default to No
            _ => {
                eprintln!("Invalid input. Please enter 'y' or 'n'.");
            }
        }
    }
}

fn main() -> io::Result<()> {
    if ask_confirmation("Do you want to proceed?")? {
        println!("Proceeding with the operation...");
    } else {
        println!("Operation cancelled.");
    }
    Ok(())
}
```

### Multiple Choice Selection

```rust
use std::io::{self, Write};

fn select_option(prompt: &str, options: &[&str]) -> io::Result<usize> {
    println!("{}", prompt);
    for (i, option) in options.iter().enumerate() {
        println!("  {}. {}", i + 1, option);
    }

    let mut input = String::new();
    loop {
        print!("Enter your choice (1-{}): ", options.len());
        io::stdout().flush()?;

        input.clear();
        io::stdin().read_line(&mut input)?;

        match input.trim().parse::<usize>() {
            Ok(n) if n >= 1 && n <= options.len() => return Ok(n - 1),
            _ => eprintln!("Invalid selection. Please try again."),
        }
    }
}

fn main() -> io::Result<()> {
    let options = ["Development", "Staging", "Production"];
    let choice = select_option("Select environment:", &options)?;
    println!("Selected: {}", options[choice]);
    Ok(())
}
```

### Reusable Input Helpers

```rust
use std::io::{self, Write};

fn prompt(msg: &str) -> io::Result<String> {
    print!("{msg}");
    io::stdout().flush()?;
    let mut input = String::new();
    io::stdin().read_line(&mut input)?;
    Ok(input.trim().to_string())
}

fn confirm(msg: &str) -> io::Result<bool> {
    loop {
        let input = prompt(&format!("{msg} (y/N): "))?;
        match input.to_lowercase().as_str() {
            "y" | "yes" => return Ok(true),
            "n" | "no" | "" => return Ok(false),
            _ => eprintln!("Please enter 'y' or 'n'."),
        }
    }
}

fn read_number(msg: &str) -> io::Result<u32> {
    loop {
        let input = prompt(msg)?;
        match input.parse() {
            Ok(n) => return Ok(n),
            Err(_) => eprintln!("Invalid number."),
        }
    }
}
```

## Styled Output with colored

```rust
// Cargo.toml: colored = "2"
use colored::*;

fn main() {
    // Basic colors
    println!("{}", "This is red".red());
    println!("{}", "This is green".green());
    println!("{}", "This is blue".blue());
    println!("{}", "This is yellow".yellow());
    println!("{}", "This is cyan".cyan());
    println!("{}", "This is magenta".magenta());

    // Styles
    println!("{}", "This is bold".bold());
    println!("{}", "This is italic".italic());
    println!("{}", "This is underline".underline());
    println!("{}", "This is dimmed".dimmed());

    // Combinations
    println!("{}", "Bold red text".red().bold());
    println!("{}", "Green on blue background".green().on_blue());
    println!("{}", "Bright cyan".bright_cyan());

    // Practical examples
    println!("{} Operation completed", "✓".green().bold());
    println!("{} Warning: check configuration", "⚠".yellow().bold());
    println!("{} Error: file not found", "✗".red().bold());
}

// Status messages helper
fn print_status(status: &str, message: &str) {
    match status {
        "success" => println!("{} {}", "[OK]".green().bold(), message),
        "warning" => println!("{} {}", "[WARN]".yellow().bold(), message),
        "error" => println!("{} {}", "[ERROR]".red().bold(), message),
        "info" => println!("{} {}", "[INFO]".blue().bold(), message),
        _ => println!("{}", message),
    }
}

// Disable colors when piping or with --no-color flag
// colored::control::set_override(false);
```

## Tables with prettytable-rs

```rust
// Cargo.toml: prettytable-rs = "0.10"
use prettytable::{Table, Row, Cell, row, format};

fn main() {
    // Basic table
    let mut table = Table::new();

    // Add header row
    table.add_row(row!["Name", "Size", "Modified"]);

    // Add data rows
    table.add_row(row!["document.txt", "1.2 KB", "2024-01-15"]);
    table.add_row(row!["image.png", "50 KB", "2024-01-14"]);
    table.add_row(row!["archive.zip", "1 MB", "2024-01-13"]);

    table.printstd();

    // Styled table with box-drawing characters
    let mut styled_table = Table::new();
    styled_table.set_format(*format::consts::FORMAT_BOX_CHARS);

    // Styled header with bold and color
    styled_table.add_row(Row::new(vec![
        Cell::new("ID").style_spec("bFc"),      // bold, foreground cyan
        Cell::new("Status").style_spec("bFc"),
        Cell::new("Description").style_spec("bFc"),
    ]));

    // Data with conditional styling
    styled_table.add_row(Row::new(vec![
        Cell::new("001"),
        Cell::new("Active").style_spec("Fg"),   // foreground green
        Cell::new("Main server"),
    ]));

    styled_table.add_row(Row::new(vec![
        Cell::new("002"),
        Cell::new("Warning").style_spec("Fy"),  // foreground yellow
        Cell::new("High memory usage"),
    ]));

    styled_table.add_row(Row::new(vec![
        Cell::new("003"),
        Cell::new("Error").style_spec("Fr"),    // foreground red
        Cell::new("Connection failed"),
    ]));

    styled_table.printstd();
}

// Dynamic table from data
fn print_users(users: &[(u32, &str, &str)]) {
    let mut table = Table::new();
    table.set_format(*format::consts::FORMAT_CLEAN);

    table.add_row(row![bFc => "ID", "Username", "Email"]);

    for (id, username, email) in users {
        table.add_row(row![id, username, email]);
    }

    table.printstd();
}
```

## Progress Indicators with indicatif

### Basic Progress Bar

```rust
// Cargo.toml: indicatif = "0.17"
use indicatif::{ProgressBar, ProgressStyle};
use std::thread;
use std::time::Duration;

fn main() {
    let pb = ProgressBar::new(100);

    pb.set_style(
        ProgressStyle::default_bar()
            .template("{msg} [{bar:40.cyan/blue}] {pos}/{len} ({eta})")
            .unwrap()
            .progress_chars("=>-"),
    );

    for i in 0..100 {
        thread::sleep(Duration::from_millis(50));
        pb.set_message(format!("Processing item {}", i + 1));
        pb.inc(1);
    }

    pb.finish_with_message("Task completed!");
}
```

### Spinner for Indeterminate Progress

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::thread;
use std::time::Duration;

fn main() {
    let spinner = ProgressBar::new_spinner();

    spinner.set_style(
        ProgressStyle::default_spinner()
            .tick_chars("⠁⠂⠄⡀⢀⠠⠐⠈ ")
            .template("{spinner:.green} {msg}")
            .unwrap(),
    );

    spinner.set_message("Connecting to server...");

    for _ in 0..50 {
        spinner.tick();
        thread::sleep(Duration::from_millis(100));
    }

    spinner.finish_with_message("Connected!");
}
```

### Multiple Progress Bars

```rust
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};
use std::thread;
use std::time::Duration;

fn main() {
    let multi = MultiProgress::new();

    let style = ProgressStyle::default_bar()
        .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos:>7}/{len:7} {msg}")
        .unwrap()
        .progress_chars("##-");

    let pb1 = multi.add(ProgressBar::new(100));
    pb1.set_style(style.clone());
    pb1.set_message("Downloading...");

    let pb2 = multi.add(ProgressBar::new(100));
    pb2.set_style(style.clone());
    pb2.set_message("Processing...");

    let pb3 = multi.add(ProgressBar::new(100));
    pb3.set_style(style);
    pb3.set_message("Uploading...");

    // Simulate concurrent tasks
    let handles: Vec<_> = [pb1, pb2, pb3]
        .into_iter()
        .map(|pb| {
            thread::spawn(move || {
                for _ in 0..100 {
                    thread::sleep(Duration::from_millis(30 + rand::random::<u64>() % 50));
                    pb.inc(1);
                }
                pb.finish_with_message("Done");
            })
        })
        .collect();

    for handle in handles {
        handle.join().unwrap();
    }
}
```

### Progress with Download-Style Output

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

fn main() {
    let total_size: u64 = 1024 * 1024 * 50; // 50 MB
    let pb = ProgressBar::new(total_size);

    pb.set_style(
        ProgressStyle::default_bar()
            .template(
                "{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] \
                 {bytes}/{total_bytes} ({bytes_per_sec}, {eta})",
            )
            .unwrap()
            .progress_chars("#>-"),
    );

    let mut downloaded: u64 = 0;
    while downloaded < total_size {
        let chunk: u64 = 1024 * 100; // 100 KB chunks
        downloaded += chunk;
        pb.set_position(downloaded);
        std::thread::sleep(Duration::from_millis(50));
    }

    pb.finish_with_message("Download complete");
}
```

## Terminal Manipulation with crossterm

### Basic Terminal Control

```rust
// Cargo.toml: crossterm = "0.27"
use crossterm::{
    cursor::{MoveTo, Hide, Show},
    execute,
    style::{Color, Print, SetForegroundColor, ResetColor},
    terminal::{Clear, ClearType, size},
};
use std::io::{stdout, Write};

fn main() -> crossterm::Result<()> {
    let mut stdout = stdout();

    // Get terminal size
    let (cols, rows) = size()?;
    println!("Terminal size: {}x{}", cols, rows);

    // Clear screen
    execute!(stdout, Clear(ClearType::All))?;

    // Move cursor and print colored text
    execute!(
        stdout,
        MoveTo(10, 5),
        SetForegroundColor(Color::Green),
        Print("Hello from position (10, 5)!"),
        ResetColor
    )?;

    execute!(
        stdout,
        MoveTo(10, 7),
        SetForegroundColor(Color::Yellow),
        Print("This is yellow text"),
        ResetColor
    )?;

    stdout.flush()?;
    Ok(())
}
```

### Alternate Screen Buffer

```rust
use crossterm::{
    execute,
    terminal::{EnterAlternateScreen, LeaveAlternateScreen, Clear, ClearType},
    cursor::MoveTo,
    style::Print,
};
use std::io::{stdout, stdin, Write, BufRead};

fn main() -> crossterm::Result<()> {
    let mut stdout = stdout();

    // Enter alternate screen (preserves user's terminal)
    execute!(stdout, EnterAlternateScreen)?;
    execute!(stdout, Clear(ClearType::All))?;

    execute!(
        stdout,
        MoveTo(0, 0),
        Print("Welcome to the alternate screen!")
    )?;

    execute!(
        stdout,
        MoveTo(0, 2),
        Print("Press Enter to exit...")
    )?;

    stdout.flush()?;

    // Wait for user input
    let mut line = String::new();
    stdin().lock().read_line(&mut line)?;

    // Leave alternate screen (restores original content)
    execute!(stdout, LeaveAlternateScreen)?;

    println!("Back to normal terminal!");
    Ok(())
}
```

### Raw Mode for Key Input

```rust
use crossterm::{
    event::{self, Event, KeyCode, KeyEvent},
    terminal::{enable_raw_mode, disable_raw_mode},
};
use std::io::{stdout, Write};

fn main() -> crossterm::Result<()> {
    enable_raw_mode()?;

    println!("Press 'q' to quit, any other key to see its code.\r");

    loop {
        if event::poll(std::time::Duration::from_millis(100))? {
            if let Event::Key(KeyEvent { code, .. }) = event::read()? {
                match code {
                    KeyCode::Char('q') => {
                        println!("\r\nExiting...\r");
                        break;
                    }
                    KeyCode::Char(c) => {
                        println!("You pressed: '{}'\r", c);
                    }
                    KeyCode::Enter => println!("You pressed: Enter\r"),
                    KeyCode::Esc => println!("You pressed: Escape\r"),
                    KeyCode::Up => println!("Arrow Up\r"),
                    KeyCode::Down => println!("Arrow Down\r"),
                    KeyCode::Left => println!("Arrow Left\r"),
                    KeyCode::Right => println!("Arrow Right\r"),
                    _ => println!("Key: {:?}\r", code),
                }
                stdout().flush()?;
            }
        }
    }

    disable_raw_mode()?;
    Ok(())
}
```

## File System Operations

### Reading and Writing Files

```rust
use std::fs::{self, File, OpenOptions};
use std::io::{self, Read, Write, BufReader, BufRead};

// Read entire file to string
fn read_file_content(path: &str) -> io::Result<String> {
    fs::read_to_string(path)
}

// Read file to bytes
fn read_file_bytes(path: &str) -> io::Result<Vec<u8>> {
    fs::read(path)
}

// Write string to file (creates or truncates)
fn write_file(path: &str, content: &str) -> io::Result<()> {
    fs::write(path, content)
}

// Write with explicit file handle
fn write_file_explicit(path: &str, content: &str) -> io::Result<()> {
    let mut file = File::create(path)?;
    file.write_all(content.as_bytes())?;
    Ok(())
}

// Read file line by line (memory efficient for large files)
fn read_lines(path: &str) -> io::Result<Vec<String>> {
    let file = File::open(path)?;
    let reader = BufReader::new(file);
    reader.lines().collect()
}

// Append to file
fn append_to_file(path: &str, content: &str) -> io::Result<()> {
    let mut file = OpenOptions::new()
        .append(true)
        .create(true)
        .open(path)?;
    writeln!(file, "{}", content)?;
    Ok(())
}
```

### Directory Operations

```rust
use std::fs;
use std::io;
use std::path::Path;

// Create directory (fails if parent doesn't exist)
fn create_dir(path: &str) -> io::Result<()> {
    fs::create_dir(path)
}

// Create directory and all parent directories
fn create_dir_recursive(path: &str) -> io::Result<()> {
    fs::create_dir_all(path)  // Succeeds even if dir exists
}

// List directory contents
fn list_directory(path: &str) -> io::Result<()> {
    for entry in fs::read_dir(path)? {
        let entry = entry?;
        let path = entry.path();
        let file_type = entry.file_type()?;

        let type_str = if file_type.is_dir() {
            "DIR "
        } else if file_type.is_file() {
            "FILE"
        } else {
            "LINK"
        };

        println!("{} {:?}", type_str, path.file_name().unwrap());
    }
    Ok(())
}

// Remove file
fn remove_file(path: &str) -> io::Result<()> {
    fs::remove_file(path)
}

// Remove empty directory
fn remove_dir(path: &str) -> io::Result<()> {
    fs::remove_dir(path)
}

// Remove directory and all contents (recursive)
fn remove_dir_recursive(path: &str) -> io::Result<()> {
    fs::remove_dir_all(path)
}

// Copy, rename
fn file_operations() -> io::Result<()> {
    fs::copy("src.txt", "dst.txt")?;
    fs::rename("old.txt", "new.txt")?;
    Ok(())
}

// Temp files (requires tempfile crate)
fn with_temp_dir() -> io::Result<()> {
    let dir = tempfile::tempdir()?;
    let file_path = dir.path().join("temp.txt");
    fs::write(&file_path, "temporary data")?;
    // dir is deleted when dropped
    Ok(())
}
```

### Path Manipulation

```rust
use std::path::{Path, PathBuf};

fn path_operations() {
    // Create PathBuf from string
    let mut path = PathBuf::from("/home/user");

    // Join path components (platform-aware separator)
    path.push("documents");
    path.push("file.txt");
    println!("Joined: {}", path.display());  // /home/user/documents/file.txt

    // Get components
    if let Some(parent) = path.parent() {
        println!("Parent: {}", parent.display());
    }
    if let Some(file_name) = path.file_name() {
        println!("Filename: {:?}", file_name);
    }
    if let Some(extension) = path.extension() {
        println!("Extension: {:?}", extension);
    }
    if let Some(stem) = path.file_stem() {
        println!("Stem: {:?}", stem);
    }

    // Check path properties
    let p = Path::new("/tmp/test.txt");
    println!("Exists: {}", p.exists());
    println!("Is file: {}", p.is_file());
    println!("Is dir: {}", p.is_dir());
    println!("Is absolute: {}", p.is_absolute());

    // Convert to string (may fail for non-UTF8 paths)
    if let Some(s) = path.to_str() {
        println!("As string: {}", s);
    }
}

// Build output path from input
fn derive_output_path(input: &Path, suffix: &str, new_ext: &str) -> PathBuf {
    let stem = input.file_stem().unwrap_or_default();
    let mut output = input.with_file_name(format!(
        "{}{}",
        stem.to_string_lossy(),
        suffix,
    ));
    output.set_extension(new_ext);
    output
}
```

### File Metadata

```rust
use std::fs;
use std::io;
use std::time::SystemTime;

fn get_file_info(path: &str) -> io::Result<()> {
    let metadata = fs::metadata(path)?;

    println!("Size: {} bytes", metadata.len());
    println!("Is file: {}", metadata.is_file());
    println!("Is directory: {}", metadata.is_dir());
    println!("Is symlink: {}", metadata.is_symlink());
    println!("Readonly: {}", metadata.permissions().readonly());

    if let Ok(modified) = metadata.modified() {
        if let Ok(duration) = modified.duration_since(SystemTime::UNIX_EPOCH) {
            println!("Modified: {} seconds since epoch", duration.as_secs());
        }
    }

    Ok(())
}

// Check if file exists before reading
fn safe_read(path: &str) -> io::Result<String> {
    let p = std::path::Path::new(path);
    if !p.exists() {
        return Err(io::Error::new(
            io::ErrorKind::NotFound,
            format!("File not found: {}", path),
        ));
    }
    if !p.is_file() {
        return Err(io::Error::new(
            io::ErrorKind::InvalidInput,
            format!("Not a file: {}", path),
        ));
    }
    fs::read_to_string(path)
}
```

### Error Handling with Context (anyhow)

```rust
// Cargo.toml: anyhow = "1.0"
use anyhow::{Context, Result};
use std::fs;
use std::path::Path;

fn process_config_file(path: &str) -> Result<Config> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("Failed to read config file: {}", path))?;

    let config: Config = toml::from_str(&content)
        .with_context(|| format!("Failed to parse config file: {}", path))?;

    Ok(config)
}

fn copy_with_context(src: &Path, dst: &Path) -> Result<()> {
    fs::copy(src, dst).with_context(|| {
        format!("Failed to copy '{}' to '{}'", src.display(), dst.display())
    })?;
    Ok(())
}

fn create_output_dir(path: &Path) -> Result<()> {
    fs::create_dir_all(path).with_context(|| {
        format!("Failed to create output directory: {}", path.display())
    })?;
    Ok(())
}
```

## Signal Handling

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    let ctrl_c = signal::ctrl_c();

    tokio::select! {
        _ = ctrl_c => {
            eprintln!("\nReceived Ctrl+C, shutting down...");
            cleanup();
        }
        _ = run_app() => {}
    }
}

// Or with ctrlc crate (synchronous)
fn main_sync() {
    ctrlc::set_handler(|| {
        eprintln!("\nInterrupted!");
        std::process::exit(1);
    })
    .expect("Error setting Ctrl-C handler");

    run_app_sync();
}
```

## Complete CLI Application Example

```rust
use clap::{Parser, Subcommand};
use colored::*;
use indicatif::{ProgressBar, ProgressStyle};
use prettytable::{Table, row, format};
use std::io::{self, Write};
use std::path::PathBuf;
use std::thread;
use std::time::Duration;

#[derive(Parser)]
#[command(name = "taskctl")]
#[command(author, version, about = "Task management CLI")]
struct Cli {
    /// Configuration file path
    #[arg(short, long, global = true)]
    config: Option<PathBuf>,

    /// Enable verbose output
    #[arg(short, long, global = true, action)]
    verbose: bool,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Add a new task
    Add {
        /// Task description
        description: String,

        /// Task priority (1-5)
        #[arg(short, long, default_value_t = 3, value_parser = clap::value_parser!(u8).range(1..=5))]
        priority: u8,
    },

    /// List all tasks
    List {
        /// Filter by status
        #[arg(short, long)]
        status: Option<String>,

        /// Output format
        #[arg(long, value_enum, default_value = "table")]
        format: OutputFormat,
    },

    /// Complete a task
    Complete {
        /// Task ID
        id: u32,
    },

    /// Delete a task
    Delete {
        /// Task ID
        id: u32,

        /// Skip confirmation
        #[arg(short, long, action)]
        force: bool,
    },

    /// Import tasks from file
    Import {
        /// File to import
        file: PathBuf,
    },
}

#[derive(Debug, Clone, clap::ValueEnum)]
enum OutputFormat {
    Table,
    Json,
    Simple,
}

fn main() -> io::Result<()> {
    let cli = Cli::parse();

    if cli.verbose {
        println!("{} Verbose mode enabled", "[DEBUG]".dimmed());
    }

    if let Some(config) = &cli.config {
        println!("{} Using config: {:?}", "[INFO]".blue(), config);
    }

    match cli.command {
        Commands::Add { description, priority } => {
            add_task(&description, priority);
        }
        Commands::List { status, format } => {
            list_tasks(status.as_deref(), &format);
        }
        Commands::Complete { id } => {
            complete_task(id);
        }
        Commands::Delete { id, force } => {
            delete_task(id, force)?;
        }
        Commands::Import { file } => {
            import_tasks(&file);
        }
    }

    Ok(())
}

fn add_task(description: &str, priority: u8) {
    let priority_color = match priority {
        1 => "low".green(),
        2 => "normal".blue(),
        3 => "medium".yellow(),
        4 => "high".red(),
        5 => "critical".red().bold(),
        _ => "unknown".normal(),
    };

    println!(
        "{} Added task: \"{}\" (priority: {})",
        "✓".green().bold(),
        description,
        priority_color,
    );
}

fn list_tasks(status: Option<&str>, format: &OutputFormat) {
    let tasks = vec![
        (1, "Complete project report", "pending", 4),
        (2, "Review pull requests", "in_progress", 3),
        (3, "Update documentation", "pending", 2),
        (4, "Fix login bug", "completed", 5),
    ];

    let filtered: Vec<_> = tasks
        .iter()
        .filter(|(_, _, s, _)| status.map_or(true, |f| *s == f))
        .collect();

    match format {
        OutputFormat::Table => {
            let mut table = Table::new();
            table.set_format(*format::consts::FORMAT_BOX_CHARS);

            table.add_row(row![bFc => "ID", "Description", "Status", "Priority"]);

            for (id, desc, status, priority) in &filtered {
                let status_cell = match *status {
                    "completed" => status.green().to_string(),
                    "in_progress" => status.yellow().to_string(),
                    "pending" => status.blue().to_string(),
                    _ => status.to_string(),
                };

                table.add_row(row![id, desc, status_cell, priority]);
            }

            table.printstd();
        }
        OutputFormat::Json => {
            println!("[");
            for (i, (id, desc, status, priority)) in filtered.iter().enumerate() {
                let comma = if i < filtered.len() - 1 { "," } else { "" };
                println!(
                    "  {{\"id\": {}, \"description\": \"{}\", \"status\": \"{}\", \"priority\": {}}}{}",
                    id, desc, status, priority, comma,
                );
            }
            println!("]");
        }
        OutputFormat::Simple => {
            for (id, desc, status, priority) in &filtered {
                println!("[{}] {} ({}, P{})", id, desc, status, priority);
            }
        }
    }
}

fn complete_task(id: u32) {
    let spinner = ProgressBar::new_spinner();
    spinner.set_style(
        ProgressStyle::default_spinner()
            .tick_chars("⠁⠂⠄⡀⢀⠠⠐⠈ ")
            .template("{spinner:.green} {msg}")
            .unwrap(),
    );

    spinner.set_message(format!("Completing task {}...", id));

    for _ in 0..20 {
        spinner.tick();
        thread::sleep(Duration::from_millis(50));
    }

    spinner.finish_with_message(format!(
        "{} Task {} marked as completed!",
        "✓".green().bold(),
        id,
    ));
}

fn delete_task(id: u32, force: bool) -> io::Result<()> {
    if !force {
        print!(
            "{} Are you sure you want to delete task {}? (y/N): ",
            "⚠".yellow().bold(),
            id,
        );
        io::stdout().flush()?;

        let mut input = String::new();
        io::stdin().read_line(&mut input)?;

        if !matches!(input.trim().to_lowercase().as_str(), "y" | "yes") {
            println!("{} Deletion cancelled.", "✗".red());
            return Ok(());
        }
    }

    println!("{} Task {} deleted.", "✓".green().bold(), id);
    Ok(())
}

fn import_tasks(file: &PathBuf) {
    println!("{} Importing from {:?}...", "[INFO]".blue(), file);

    let pb = ProgressBar::new(100);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("{msg} [{bar:40.cyan/blue}] {pos}%")
            .unwrap()
            .progress_chars("=>-"),
    );

    for i in 0..=100 {
        pb.set_position(i);
        pb.set_message("Processing");
        thread::sleep(Duration::from_millis(20));
    }

    pb.finish_with_message("Import complete");
    println!("{} Imported 42 tasks from file.", "✓".green().bold());
}
```

## Graceful Exit Codes (ExitCode Pattern)

Never use `std::process::exit()` — it skips destructors and leaves resources uncleaned. Return `ExitCode` from `main()` instead (pattern from ripgrep):

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(true) => ExitCode::SUCCESS,    // Found results / success
        Ok(false) => ExitCode::from(1),    // No results (grep convention)
        Err(err) => {
            // Check for broken pipe — exit silently (Unix convention)
            if is_broken_pipe(&err) {
                return ExitCode::SUCCESS;
            }
            eprintln!("Error: {err:#}");
            ExitCode::from(2)              // Actual error
        }
    }
}

fn is_broken_pipe(err: &anyhow::Error) -> bool {
    for cause in err.chain() {
        if let Some(io_err) = cause.downcast_ref::<std::io::Error>() {
            if io_err.kind() == std::io::ErrorKind::BrokenPipe {
                return true;
            }
        }
    }
    false
}

fn run() -> anyhow::Result<bool> {
    // All application logic here — returns Result
    // ExitCode mapping happens only in main()
    Ok(true)
}
```

**Exit code conventions:**

| Code | Meaning | Example |
|------|---------|---------|
| 0 | Success | Found results, operation completed |
| 1 | No results / soft failure | grep found no matches |
| 2 | Error | Invalid arguments, I/O failure |

## Terminal Detection (`IsTerminal`)

Detect interactive vs piped output to disable colors, progress bars, and prompts:

```rust
use std::io::IsTerminal;

fn main() {
    // std::io::IsTerminal — stable since Rust 1.70 (replaces deprecated atty crate)
    let interactive = std::io::stdout().is_terminal();

    if !interactive {
        // Piped to another program — disable colors and progress
        colored::control::set_override(false);
    }

    // Conditionally show progress bars
    if interactive {
        let pb = indicatif::ProgressBar::new(100);
        // ... show progress
    } else {
        // Silent — output only data
    }
}
```

## Argument Conflicts and Requirements

```rust
use clap::{Parser, ArgGroup};

#[derive(Parser)]
#[command(group(ArgGroup::new("output")
    .required(false)
    .args(["json", "csv", "table"])))]
struct Args {
    /// Output as JSON
    #[arg(long, group = "output")]
    json: bool,

    /// Output as CSV
    #[arg(long, group = "output")]
    csv: bool,

    /// Output as table (default)
    #[arg(long, group = "output")]
    table: bool,

    /// Verbose mode — conflicts with quiet
    #[arg(short, long, conflicts_with = "quiet")]
    verbose: bool,

    /// Quiet mode — conflicts with verbose
    #[arg(short, long, conflicts_with = "verbose")]
    quiet: bool,

    /// Output file — requires format flag
    #[arg(short, long, requires = "output")]
    output_file: Option<String>,
}
```

## Clap 4.x Custom Styles

Customize help output appearance with the `styles` API:

```rust
use clap::builder::styling::{AnsiColor, Effects, Styles};

const STYLES: Styles = Styles::styled()
    .header(AnsiColor::Yellow.on_default() | Effects::BOLD)
    .usage(AnsiColor::Yellow.on_default() | Effects::BOLD)
    .literal(AnsiColor::Green.on_default() | Effects::BOLD)
    .placeholder(AnsiColor::Cyan.on_default());

#[derive(clap::Parser)]
#[command(styles = STYLES, about = "My styled CLI tool")]
struct Cli {
    #[arg(short, long)]
    verbose: bool,
}
```

## Best Practices

| Practice | Description |
|----------|-------------|
| **Help text** | Always provide clear `--help` output with examples |
| **Exit codes** | Return `ExitCode` from `main()` — never call `process::exit()` |
| **stderr for errors** | Use `eprintln!` for errors, `println!` for normal output |
| **Color fallback** | Use `std::io::IsTerminal` + `colored::control::set_override(false)` |
| **Progress for long ops** | Always show progress for operations > 1 second |
| **Confirmation for destructive** | Require `--force` or confirmation for destructive operations |
| **Config precedence** | CLI args > env vars > config file > defaults |
| **Pipe-friendly** | Detect `stdout().is_terminal()` and disable colors/progress when piping |
| **Broken pipe** | Walk error chain for `BrokenPipe`, exit with code 0 |

## Crate Summary

| Crate | Purpose |
|-------|---------|
| `clap` | Argument parsing (derive + builder) |
| `indicatif` | Progress bars, spinners |
| `colored` | Terminal colors and styles |
| `prettytable-rs` | Formatted tables |
| `comfy-table` | Alternative table formatter |
| `crossterm` | Terminal manipulation, raw mode, events |
| `dialoguer` | Interactive prompts, selections, fuzzy matching |
| `console` | Terminal abstraction (by indicatif author) |
| `tempfile` | Temporary files and directories |
| `assert_cmd` | CLI integration testing |
| `ctrlc` | Signal handling (sync) |
| `anyhow` | Error handling with context |

## Related Skills

- **[SKILL.md](SKILL.md)** — Core Rust: error handling, serde, iterators, pattern matching
- **[error-handling.md](error-handling.md)** — `color-eyre` for CLI error display, `anyhow` context chains
- **[serde-serialization.md](serde-serialization.md)** — Config file parsing, TOML/JSON/YAML deserialization
- **[testing.md](testing.md)** — `assert_cmd` for CLI integration testing, `tempfile` for fixtures
