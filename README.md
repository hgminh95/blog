# blog

Simple blog written in hugo. Check out https://blog.hgminh.dev

## Usage

To run the server locally

```bash
$ hugo server -D
```

To build it, run

```bash
$ HUGO_ENV=production hugo
```

Note that we add the build artifact to git since Cloudflare Pages does not support newer version of hugo.
