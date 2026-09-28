# 2D DASH server preparation

This directory prepares one normal video for the 2D DASH client.  It downloads
or accepts a source video, encodes several video qualities, and generates a
static MPD plus fragmented MP4 segments.  It does not implement an HTTP server;
`nghttpd` serves the generated files during the end-to-end experiment.

---

## I. Directory structure

```
2d_DASH_server/
├── prepare_dash.sh    # Download, encode, and package the DASH content
└── README.md          # This guide
```

The script produces a separate output directory:

```
dash_output/
├── manifest.mpd
├── init-0.m4s
├── init-1.m4s
├── init-2.m4s
└── chunk-<representation>-<number>.m4s
```

---

## II. Requirements

On Ubuntu/Debian, install the usual tools with:

```sh
sudo apt install curl ffmpeg nghttp2-server openssl
```

The script chooses `libx264` when FFmpeg provides it; otherwise it falls back
to FFmpeg's built-in `mpeg4` encoder. This fallback keeps the lab portable,
but H.264 is normally the more compatible choice for real DASH players.
`nghttp2-server` provides the `nghttpd` HTTP/2 server used below.

---

## III. Create DASH content

From this directory, run either command below.  The output directory must not
already exist, so an old experiment cannot be overwritten accidentally.

```sh
# Download a source video first.
bash prepare_dash.sh \
  --input-url 'https://example.com/source.mp4' \
  --output-dir ../dash_output \
  --segment-duration 2

# Or package an existing local video.
bash prepare_dash.sh \
  --input-file ../source.mp4 \
  --output-dir ../dash_output \
  --segment-duration 2
```

Use `--video-codec libx264`, `--video-codec libx265`, or
`--video-codec mpeg4` to force an encoder. `--video-codec auto` is the
default and selects `libx264` first, then `mpeg4` when x264 is unavailable.

The script creates 270p, 360p, and 720p representations. It uses a
static `SegmentTemplate` MPD and `$Number%05d$` media-segment naming, so the
first client exercise does not need `SegmentTimeline` or `$Time$` parsing.

### Test the preparation module

Check the script syntax and its command-line interface before encoding:

```sh
bash -n prepare_dash.sh
bash prepare_dash.sh --help
```

After a successful encoding, verify that the important output exists:

```sh
test -s ../dash_output/manifest.mpd
grep -F 'SegmentTemplate' ../dash_output/manifest.mpd
find ../dash_output -name '*.m4s' -type f
```

---

## IV. Serve the content with HTTP/2

Create a short-lived local certificate for the lab.  Do not use this
self-signed certificate in a real deployment.

```sh
openssl req -x509 -newkey rsa:2048 -nodes -days 1 \
  -keyout local.key -out local.crt -subj '/CN=localhost'

nghttpd -d ../dash_output 8443 local.key local.crt
```

Leave `nghttpd` running in this terminal.  In another terminal, verify that
the manifest is reachable through HTTP/2:

```sh
curl -k --http2 -I https://localhost:8443/manifest.mpd
```

The client guide explains how to run the parser check and the complete
streaming experiment against this server.
