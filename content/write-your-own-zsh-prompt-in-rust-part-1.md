+++
title = "# Writing Your Own ZSH Prompt in Rust Pt.1: Basic Prompt"
date = "2025-03-19"

[taxonomies]
tags = ["rust", "zsh", "prompt"]
+++



Hola! If you're here, you probably want to write your own ZSH prompt in Rust. Great! I was in the same position, inspired by great prompts like [Starship](https://starship.rs/) and [Spaceship](https://github.com/spaceship-prompt/spaceship-prompt), so I decided to make my own in Rust. It's highly inspired by Starship — almost a clone — but I wanted to build it myself. That’s how I learn: by doing. Since I’m currently learning Rust, the language choice was obvious.

## Why Rust?

The first time I learned Rust was several years ago—maybe five or six—and I instantly fell in love with its structure. The C-like syntax (which I’ve always liked), strict memory management, structs, generics, traits, and other cool features made it stand out.

However, after some time, I switched to JavaScript for work and almost abandoned Rust. But recently, something happened that made me return to it. I felt the urge to do some engineering and nerdy projects again—so here I am, back in Rust, building something fun!

## What We Need

Let's list the things we need to make our prompt:

- [ZSH](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH)
- [Rust](https://www.rust-lang.org/tools/install)

Yep, just these two things. The choice of editor/IDE doesn’t matter.

## How It Works

The process is simple: in the ZSH config, we call our binary, which generates the prompt that ZSH uses.

Let's name our project **Lazer** and create a new Rust project:

```bash
cargo new lazer
```

The output will look like this:

```bash
    Creating binary (application) `lazer-article` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
```

Now, let’s open `src/main.rs`. By default, we’ll see this code:

```rust
fn main() {
    println!("Hello, world!");
}
```

### Adding Our Binary to ZSH

ZSH allows us to customize the prompt using the `precmd()` hook in the `~/.zshrc` config. First, let’s create a basic prompt setting in our code:

```rust
fn main() {
    let precmd_hook = "precmd() {
PROMPT=\"lazer > \"
}";

    println!("{precmd_hook}");
}
```

Now, we need to build our Rust project:

```bash
cargo build
```

After running the build command, you’ll find a `debug` folder inside the `target` directory, containing our binary called `lazer`. That’s what we need.

Next, we need to let ZSH know where our binary is located. Add this alias at the end of your `~/.zshrc` config:

```zsh
alias lazer=<your-project-location>/target/debug/lazer
```

Now, we need to call our binary so that it sets the prompt in ZSH.

Add this line to the end of `~/.zshrc`:

```zsh
eval "$(lazer)"
```

This evaluates the output from our binary. If we restart ZSH by running `zsh` in the terminal, we should see our prompt:

```
lazer >
```

Great! Our prompt works!

## Organizing the Code

Right now, editing the `precmd()` hook directly in Rust isn’t very convenient. To keep our code clean and modular, we’ll move the prompt setup into a separate file. Inside the src folder, create a new file called init.zsh and add the following code:

```zsh
precmd() {
    PROMPT="$(lazer prompt)"
}
```

#### What This Does:

- `precmd()` – This is a ZSH hook that runs before every command, ensuring our prompt is updated dynamically.
- `PROMPT="$(lazer prompt)"` – This sets the prompt by calling our Rust program (`lazer prompt` will be introduced soon), which returns the prompt string.

Then, in `main.rs`, we use `include_str!` to embed this file’s contents:

```rust
const ZSH_INIT: &str = include_str!("./init.zsh");
```

In Rust, `include_str!` is a macro that allows you to embed the contents of an external file as a string at compile time. This means that instead of reading the file dynamically at runtime, Rust includes the file’s contents directly into the compiled binary.

#### **Why Use `include_str!`?**

1. **Performance** – Since the file is embedded at compile time, there’s no need for file I/O operations when the program runs.
2. **Portability** – The program doesn’t depend on external files being present when executed.
3. **Simplicity** – You can keep configuration or script files in separate files while still accessing them as string literals in your code.

Now, whenever we reference `ZSH_INIT`, it contains the entire content of `init.zsh` as a string. Later, when we run `lazer init` (init will be introduced soon), we print this string to the terminal, allowing ZSH to source it.

### **Setting Up the CLI with Clap**

Since we want `lazer` to work as a command-line tool with subcommands to make it handful to use and debug, we’ll use [Clap](https://docs.rs/clap/latest/clap/), a powerful Rust library for building CLI applications. To install it, run:

```bash
cargo add clap
```

Now, let’s define our CLI structure in Rust:

```rust
fn cli() -> Command {
    Command::new("lazer")
        .about("The L A Z E R Prompt")
        .subcommand_required(true)
        .arg_required_else_help(true)
        .subcommand(Command::new("init").about("Outputs the ZSH initialization script"))
        .subcommand(Command::new("prompt").about("Generates the prompt string for ZSH"))
}
```

This function sets up the `lazer` command with two subcommands:

- **`init`** → Prints the ZSH initialization script, which sets up the prompt correctly.
- **`prompt`** → Outputs the actual prompt string that ZSH will use.

#### **Implementing the CLI Logic**

Now, let’s integrate these commands into our `main.rs`:

```rust
fn main() {
    let matches = cli().get_matches();

    match matches.subcommand() {
        Some(("init", _)) => {
            println!("{}", ZSH_INIT);
        }
        Some(("prompt", _)) => {
            print!("lazer > ");
        }
        _ => unreachable!(),
    }
}
```

Here’s how it works:

1. We call `cli().get_matches()` to parse the user’s input.
2. If the user runs `lazer init`, we print `ZSH_INIT`, which contains our ZSH script (embedded using `include_str!`).
3. If the user runs `lazer prompt`, we print `"lazer > "`, which will be used as the command prompt.
4. Any other command is considered invalid (`unreachable!()` ensures this won’t happen).

### **Final Step: Updating `~/.zshrc`**

The last step is to update our `~/.zshrc` file to use our new `init` command instead of directly evaluating `lazer`:

```diff
- eval "$(lazer)"
+ eval "$(lazer init)"
```

Now, restart ZSH by typing:

```bash
zsh
```

And voilà! You should see your custom prompt in action:

```
lazer >
```

### **What’s Next?**

Awesome! We’ve successfully built a basic ZSH prompt in Rust. But this is just the beginning. In the next parts of this series, we’ll take it to the next level by adding:

- **Useful Modules** – Display user, directory, Git status, and more.
- **Error Handling & Debugging** – Improve stability and make troubleshooting easier.
- **Color Support** – Give our prompt a stylish, vibrant look.
- **String Templating** – Allow users to customize the prompt layout.
- **Configuration File** – Move hardcoded settings to a TOML config for flexibility.

Stay tuned! 🚀