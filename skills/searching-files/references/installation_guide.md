# Ripgrep Installation Guide

## macOS
```
brew install ripgrep
```

## Linux

**Debian/Ubuntu:**
```
apt install ripgrep
```

**Arch Linux:**
```
pacman -S ripgrep
```

**Fedora:**
```
dnf install ripgrep
```

**openSUSE:**
```
zypper install ripgrep
```

**Generic (binary download):**
Download from https://github.com/BurntSushi/ripgrep/releases and place binary in `$PATH`.

## Windows

**winget:**
```
winget install BurntSushi.ripgrep.MSVC
```

**Chocolatey:**
```
choco install ripgrep
```

**Scoop:**
```
scoop install rg
```

## Any platform (Cargo)
```
cargo install ripgrep
```

## Verify installation
```
rg --version
```
