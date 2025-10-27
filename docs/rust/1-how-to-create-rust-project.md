---
tags:
  - rust
  - rust-project
---
This page will introduce how to create a rust project.

#### 1. binary project
we can create a simple binary project, which can build a binary output.

```shell
cargo new appName
```


#### 2. create a library project
we can create a library project, which can be used as a dependency to other project.
```shell
cargo new library --lib
```


#### 3. workspace
As the project size grows.  we may want to split the common function to another library project, then we can create a workspace, it can include multiple library project and one binary project. 

```shell
mkdir workspaceName

cd workspaceName

cargo init 

## for Cargo.toml, config as below
[workspace]
resolver = "3"

```

```shell
## add library to workspace
cd workspaceName
cargo new Libproject --lib

## create binary project
cargo new appBinary  

## config library as dependency as binary project

[dependencies]
LibProject = {path = "../Libproject"}

```













