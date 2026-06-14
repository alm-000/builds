# video-pipeline

**Theme:** AI/automation · full-stack

## Problem

Producing branded short-form video at volume (social content, product clips) requires consistent overlay composition: logo, watermark, text, aspect ratio conversion, and optional endcard concatenation. Doing this manually in a GUI is slow and non-reproducible. AI-generated FFmpeg commands are unpredictable and unsafe for production.

## Approach

A template-driven Python renderer that builds and runs FFmpeg filter graphs deterministically from JSON templates. No AI-generated commands: every `ffmpeg` invocation is constructed from a strict, validated template.

**Design choices:**

- **JSON templates** — each template specifies output dimensions, fps, bitrate, and a named set of overlay configurations (logo, watermark, text with timing). Templates are version-controlled and auditable.
- **Deterministic filter complex** — the Python code builds the FFmpeg filter chain in a fixed order from the template; the same template always produces the same command.
- **Asset validation before render** — all referenced assets (logo, watermark, font, endcard) are checked for existence before FFmpeg is called. Fails fast with a clear error rather than mid-render.
- **Aspect ratio handling** — `build_scale_filter()` selects scale/crop or scale/pad based on whether the input is wider or taller than the target, avoiding letterbox/pillarbox without distortion.
- **Endcard concatenation** — optional endcard clip is prepared at matching dimensions and concatenated via `ffmpeg -f concat`, keeping the output as a single file.

## Architecture

```python
render(input_path, output_path, template_id)
  → load_template()             # JSON: dimensions, fps, overlay configs
  → validate_assets()           # check logo/watermark/font exist
  → get_video_info()            # ffprobe: duration, width, height
  → build_scale_filter()        # scale/crop/pad to target aspect ratio
  → build_overlay_filter()      # per-overlay: timing, alpha, position
  → [ffmpeg filter_complex]     # assembled deterministically from template
  → concat endcard (optional)   # separate ffmpeg pass + concat
```

## Real results / metrics

- Renders a 60-second 1080×1920 clip with logo + watermark + text overlay in ~45 seconds on an M-series Mac (H.264, medium preset, 6M bitrate).
- Template system made brand updates (new logo, font change) a 1-minute config edit rather than a code change.
- Deterministic build: same template + same input = byte-identical filter complex string every run, making debugging straightforward.

## Snippet

Aspect-ratio-aware scale filter and the main render entry point:

```python
def build_scale_filter(
    input_width: int, input_height: int,
    output_width: int, output_height: int
) -> str:
    """
    Build scale/crop/pad filter to convert to target aspect ratio.
    Uses scale + crop for aspect ratio conversion, then pad if needed.
    """
    input_aspect  = input_width  / input_height
    output_aspect = output_width / output_height

    if abs(input_aspect - output_aspect) < 0.01:
        return f"scale={output_width}:{output_height}"
    elif input_aspect > output_aspect:
        # Input is wider — scale to height, crop width
        scale_h = output_height
        scale_w = int(input_width * (output_height / input_height))
        crop_x  = (scale_w - output_width) // 2
        return f"scale={scale_w}:{scale_h},crop={output_width}:{output_height}:{crop_x}:0"
    else:
        # Input is taller — scale to width, crop height
        scale_w = output_width
        scale_h = int(input_height * (output_width / input_width))
        crop_y  = (scale_h - output_height) // 2
        return f"scale={scale_w}:{scale_h},crop={output_width}:{output_height}:0:{crop_y}"


def render(
    input_path: str,
    output_path: str,
    template_id: str = "brand_v1",
    text_override: str | None = None,
) -> str:
    """
    Render video with branding overlays using the specified template.
    All FFmpeg commands are built from validated JSON templates — no AI-generated commands.
    """
    template       = load_template(template_id)
    assets_dir     = Path(__file__).parent / "assets" / "branding"
    resolved_assets = validate_assets(template, assets_dir)
    video_info     = get_video_info(input_path)

    # ... builds filter_complex deterministically, runs ffmpeg, optionally concatenates endcard
    return final_output
```
