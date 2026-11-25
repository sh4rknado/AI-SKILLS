---
name: podman-openscad-stl
description: Generate STL files from OpenSCAD scripts inside a rootless Podman container using volume mounts and user namespace mapping. Use when creating, updating, or running a workflow or skill that renders .scad scripts to .stl files via podman run, openscad CLI, -D parameters, or /openscad mounts.
---

# podman-openscad-stl

Generate STL files from OpenSCAD scripts inside a Podman container while preserving file ownership and SELinux/volume permissions.

## When to Use

* Rendering a `.scad` script into a `.stl` mesh using `podman run` and `docker.io/openscad/openscad:latest`.
* Passing dynamic variables (`-D`) or using a custom working directory and output path inside `/openscad`.
* Automating repeatable STL builds for 3D printing or CAD pipelines.

## Steps

1. Resolve the target directory on the host containing the source `.scad` script and mount it to `/openscad` with `:Z` so SELinux/container volume labeling works cleanly.
2. Pass `--userns=keep-id` and `--rm` on `podman run` so generated `.stl` files belong to the host user and temporary containers clean up automatically.
3. Invoke `openscad` inside the container with relative or `/openscad`-rooted paths, injecting any parametric overrides with `-D 'param=value'` when needed:

```bash
podman run --rm \
  -v "$(pwd):/openscad:Z" \
  --userns=keep-id \
  -w /openscad \
  docker.io/openscad/openscad:latest \
  openscad -o output.stl input.scad
```

Example with explicit `/openscad` output/input paths and `-D` parameters:

```bash
podman run --rm \
  -v "$(pwd):/openscad:Z" \
  --userns=keep-id \
  -w /openscad \
  docker.io/openscad/openscad:latest \
  openscad -o /openscad/out/piece.stl \
    -D 'width=45' \
    -D 'thickness=3.2' \
    /openscad/scripts/piece.scad
```

4. Verify that the command exits with status `0` and that the output `.stl` file exists and is non-empty (`test -s <file.stl>`).

## Gotchas

* Avoid backslashes immediately followed by spaces in multiline shell commands; keep `\` strictly at the end of each line before the newline.
* Watch out for nested or `use <...>` / `include <...>` libraries in `.scad` files: all referenced libraries must reside inside the mounted directory (or an additional mounted path) so OpenSCAD can resolve them inside the container.
* Quote `-D` expressions carefully so the shell does not expand `$` or glob characters before `openscad` parses them.