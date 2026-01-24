This repository contains the source code for my personal website.

## Usage

## Prerequisites

- [Task](https://taskfile.dev/) - Task runner for common operations
- [Hugo](https://gohugo.io/) - Static site generator

```bash
task --list
```

### Common Tasks

Run locally:

```bash
task serve
```

Create post:

```bash
task new:post TITLE=your-post-title
```

Create nugget:

```bash
task new:nugget TITLE=your-nugget-title
```

Build for production:

```bash
task build
```
