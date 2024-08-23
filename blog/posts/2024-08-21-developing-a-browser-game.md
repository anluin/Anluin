# 💻 Developing a Browser Game 🎮

> Delving into the Depths of C, WebAssembly and WebGL

Ever felt the urge to develop a (browser) game, only to faceplant because you have absolutely no clue what you're doing?

Yep, same here. This is yet another valiant—and likely doomed—attempt.

But hey, at least this time I'm documenting it, so future me won't have to reinvent the wheel... again. 🚀

## Project Setup 🛠️

Strap in, folks! We're diving in headfirst.

```sh
mkdir ~/projects/browsergame && cd "$_"
touch makefile
touch readme.md

mkdir src && cd "$_"
touch main.c

cd ../
```

And voilà! You should have the following directory structure:

```sh
$ tree
.
├── makefile
├── readme.md
└── src
    └── main.c

1 directory, 3 files
```

First things first, let's whip up a `main.c` file as our entry point:

```c
int main(void) {
    return 0;
}
```

Sure, our program does nada right now, but let's not get ahead of ourselves.

Time to focus on our build system. Introducing: the Makefile 🎩...

```makefile
# Project name
NAME = browsergame

# Compiler & flags
COMPILER = clang
TARGET   = wasm32-unknown-unknown
CFLAGS   = -Isrc/lib -Wall -Wextra
LDFLAGS  =

# Default build mode
BUILD ?= release

# Adjustments for WASM target
ifneq ($(findstring wasm, $(TARGET)),)
	NAME    := $(NAME).wasm
	CFLAGS  += --no-standard-libraries
	LDFLAGS += -Wl,--no-entry
endif

# Compiler flags for "release" or "debug"
ifeq (release, $(BUILD))
	CFLAGS  += -Werror -Oz -flto
	LDFLAGS += -Wl,-strip-all
else
	CFLAGS  += -O0 -g
	LDFLAGS +=
endif

# Definition of phony targets
.PHONY: all build clean

# Main target, builds everything
all: target/$(TARGET)/$(BUILD)/$(NAME)

# Clean target
clean:
	rm -rf target

# Validation of the BUILD mode
ifneq ($(filter $(BUILD),debug release),$(BUILD))
	$(error "Invalid BUILD=$(BUILD), must be 'debug' or 'release'")
endif

# Finding source and object files
SOURCE_FILES = $(shell find src -type f -name *.c)
OBJECT_FILES = $(patsubst src/%.c, target/$(TARGET)/$(BUILD)/artifacts/%.o, $(SOURCE_FILES))

# Inclusion of dependency files
include $(wildcard $(patsubst %.o, %.d, $(OBJECT_FILES)))

# Creating directories for object files
$(sort $(dir $(OBJECT_FILES))):
	@mkdir -p $@

# Rule for compiling C files
target/$(TARGET)/$(BUILD)/artifacts/%.o: src/%.c | $(sort $(dir $(OBJECT_FILES))) makefile
	@$(COMPILER) --target=$(TARGET) $(CFLAGS) -MMD -c -o $@ $< \
		&& echo "\033[1;32m[INFO]\033[0m Compiled successfully:" $@

# Rule for linking object files and creating the binary
target/$(TARGET)/$(BUILD)/$(NAME): $(OBJECT_FILES) | makefile
	@$(COMPILER) --target=$(TARGET) $(CFLAGS) $(LDFLAGS) $(LG) -o $@ $^   \
		&& echo "\033[1;32m[INFO]\033[0m Compiled successfully:" $@       \
		&& printf "\033[1;32m[INFO]\033[0m Compilation completed (Size: " \
		&& wc -c < $@ | numfmt --to=iec-i --suffix=B --format="%f)"
```

### How to Use the Makefile 📜

#### Build Modes 🎮

The Makefile supports two build modes: `debug` and `release`. By default, it's set to `release`.

#### Commands ⚙️

##### Release Build 🏁

To generate a release build, simply run:

```sh
make all
```

Or specify the release mode explicitly:

```sh
BUILD=release make all
```

##### Debug Build 🔍

To generate a debug build, use the following command:

```sh
BUILD=debug make all
```

#### Clean Build Directory 🧹

To clean up the build directory, run:

```sh
make clean
```

#### Summary 📗

- **Release Build:** `make all` or `BUILD=release make all`
- **Debug Build:** `BUILD=debug make all`
- **Clean:** `make clean`

```sh
BUILD=release make all
```

Let's check it out!

```sh
$ wasm2wat target/wasm32-unknown-unknown/release/browsergame.wasm
(module
  (memory (;0;) 2)
  (export "memory" (memory 0)))
```

Wup wup wup! A valid WebAssembly binary that does nothing except export a WebAssembly.Memory. 

It's not much, but it's a
start! 🎉

Next up, we’ll add some actual functionality. Stay tuned for the next installment where our binary does more than be a
glorified memory stick. 

Until then, happy coding! 🕹️✨
