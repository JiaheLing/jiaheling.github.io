# PhotoLab portfolio management

The PhotoLab page now supports two ways of loading photos:

1. **CSV loading** on Cloudflare Pages / GitHub Pages.
2. **Static manifest loading** through `photography/photos.js`, which also works when previewing locally.

Because static hosts do not expose folder listings to the browser, simply dropping a new image into
`photography/digital/` or `photography/film/` is not enough. The page needs either a CSV row or a generated manifest entry.

## Recommended workflow

### Digital photos

1. Put image files in `photography/digital/`.
2. Update `photography/digital/photos.csv`.

```csv
filename,device,location,description,date
tokyo-night.jpg,Fujifilm X100VI,Tokyo,Evening street scene,2025-04
```

### Film photos

1. Put image files in `photography/film/`.
2. Update `photography/film/photos.csv`.

```csv
filename,device,film,location,description,date
kyoto-portra.jpg,Rollei 35S,Kodak Portra 400,Kyoto,Temple garden,2025-03
```

### Generate the static manifest

From the site root, run:

```bash
python scripts/generate_photolab_manifest.py
```

This creates/updates:

```text
photography/photos.js
```

Then upload/push the whole site.

## Cloudflare Pages Direct Upload

Upload the full site folder after adding photos, editing CSV files, and running the generator.

## GitHub Pages / github.io

This also works on GitHub Pages because all paths are relative. Commit and push:

```text
photography/digital/*
photography/film/*
photography/digital/photos.csv
photography/film/photos.csv
photography/photos.js
```

## Notes

- `filename` must exactly match the image file name, including capitalization and extension.
- Keep commas inside quoted fields, for example `"New York, NY"`.
- Supported image formats: `.jpg`, `.jpeg`, `.png`, `.webp`, `.avif`, `.gif`.
