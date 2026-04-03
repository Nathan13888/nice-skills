# Language & Ecosystem Conventions

When writing install commands, code examples, and project setup instructions, follow the
conventions of the ecosystem. People expect familiar patterns — don't reinvent them.

---

## Python

**Install commands** (pick based on what the project actually uses):
```bash
pip install my-package              # pip (most universal)
uv add my-package                   # uv
poetry add my-package               # poetry
pdm add my-package                  # pdm
```

If the project uses uv, poetry, or pdm internally, show that first. But always mention
`pip install` as the universal fallback for end users unless the package isn't on PyPI.

**Code examples:**
- Always show the import first
- Use f-strings (not `.format()` or `%`)
- Type hints are nice in examples if the library uses them, but don't add them if the
  library doesn't
- Show `if __name__ == "__main__":` only if the example is meant to be run as a script

**Version/compatibility:** State minimum Python version. Check `python_requires` in
pyproject.toml or setup.cfg.

**Dev setup pattern:**
```bash
git clone ... && cd ...
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
```

If the project uses uv, prefer `uv sync` over manual venv creation.

---

## JavaScript / TypeScript

**Install commands** (detect from lockfile):
```bash
npm install my-package              # npm (package-lock.json)
yarn add my-package                 # yarn (yarn.lock)
pnpm add my-package                 # pnpm (pnpm-lock.yaml)
bun add my-package                  # bun (bun.lockb)
```

Show the one that matches the project's lockfile. For library READMEs consumed by others,
show `npm install` as the default since it's most universal.

**Code examples:**
- Use ESM (`import`) by default unless the project specifically targets CommonJS
- For TypeScript projects, show TypeScript examples (not JS)
- Include the `from` path — people need to know the exact import path
- For React libraries, show a JSX example

**Version/compatibility:** State minimum Node.js version. Check `engines` in package.json.

**Dev setup pattern:**
```bash
git clone ... && cd ...
npm install
npm run dev          # or npm test
```

---

## Rust

**Install commands:**
```bash
cargo install my-tool               # CLI tools (installs binary)
cargo add my-crate                  # libraries (adds to Cargo.toml)
```

For libraries, show `cargo add`. For CLI tools, show `cargo install`. If the tool is also
available via homebrew or other package managers, show those alternatives.

**Code examples:**
- Show `use` statements
- Include `fn main()` wrapper for runnable examples
- Add brief comments for non-obvious Rust patterns (lifetimes, trait bounds)
- Use `?` for error handling in examples, mention what error type is returned

**Version/compatibility:** State minimum Rust edition and MSRV (minimum supported Rust
version) if the project defines one. Check `rust-version` in Cargo.toml.

**Dev setup pattern:**
```bash
git clone ... && cd ...
cargo build
cargo test
```

---

## Go

**Install commands:**
```bash
go install github.com/user/tool@latest    # CLI tools
go get github.com/user/lib                # libraries
```

**Code examples:**
- Show the full import path
- Include `package main` and `func main()` for runnable examples
- Handle errors explicitly (don't use `_` for error returns in examples)

**Version/compatibility:** State minimum Go version. Check `go` directive in go.mod.

**Dev setup pattern:**
```bash
git clone ... && cd ...
go build ./...
go test ./...
```

---

## Ruby

**Install commands:**
```bash
gem install my-gem                  # standalone
# or in Gemfile:
gem 'my-gem', '~> 1.0'
bundle install
```

**Code examples:**
- Show `require` first
- Use modern Ruby syntax (keyword args, pattern matching if relevant)

**Dev setup pattern:**
```bash
git clone ... && cd ...
bundle install
bundle exec rake test
```

---

## Java / Kotlin

**Install commands** — show the dependency snippet for the project's build tool:

For Gradle:
```groovy
implementation 'com.example:my-lib:1.0.0'
```

For Maven:
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>my-lib</artifactId>
  <version>1.0.0</version>
</dependency>
```

Show whichever matches the project's build system. If the project uses Gradle, show Gradle.

**Dev setup pattern:**
```bash
git clone ... && cd ...
./gradlew build      # or mvn package
./gradlew test       # or mvn test
```

---

## C / C++

Install patterns vary wildly. Common approaches:

```bash
# CMake
cmake -B build && cmake --build build
sudo cmake --install build

# Make
make && sudo make install

# vcpkg / Conan
vcpkg install my-lib
conan install my-lib
```

Always mention build prerequisites (compiler version, cmake version, system libraries).

---

## General patterns (all languages)

- Show one install method prominently, put alternatives in `<details>` blocks
- Use the project's actual package/crate/gem name, not a placeholder
- If the project isn't published to a registry yet, show install-from-source only
- Match the code style of the project itself in examples (if the project uses tabs, use tabs)
- If a project uses Docker as the primary dev experience, show that as the main setup path
