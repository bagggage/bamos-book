# BamOS Documentation

This repository contains all the source files for the documentation book for the [BamOS](https://github.com/bagggage/bamos) project.

The documentation is based on [mdBook](https://github.com/rust-lang/mdBook).

## Build

To build the documentation, you need to have the `mdBook` package pre-installed.  
You can get it from the [precompiled binaries](https://github.com/rust-lang/mdBook/releases) or build it from source if you have the `rust` toolchain installed:  
```sh
cargo install mdbook
```

Then, clone the repository and build the documentation:  
```sh
git clone https://github.com/bagggage/bamos-book.git;
cd bamos-book;
mdbook build
```

## Read and Edit

To open the documentation locally:  
```sh
mdbook serve --open
```

This will start an HTML server. Additionally, `mdBook` will monitor any changes made to the documentation, rebuild the pages, and refresh them automatically.

The main content is located in the `/src` directory, where all the pages of the documentation are stored. Each page is a separate Markdown file. If you want to make changes, simply edit and save the file.