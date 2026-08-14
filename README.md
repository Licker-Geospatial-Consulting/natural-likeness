# Web map

A static Leaflet map of the nature-likeness layer. No backend: an XYZ tile pyramid,
a small binary grid for click readout, and one HTML file.

## Contents

```
index.html      the map
meta.json       zoom range, bounds, anchor set (written by 08_build_webmap.py)
tiles/{z}/{x}/{y}.png   greyscale VALUE tiles (pixel = score byte, 0 = nodata)
values.bin      coarse uint8 score grid, for the click readout
values.json     header for values.bin (grid size, Web Mercator origin, resolution)
```

Rebuild with:

```bash
uv run python scripts/08_build_webmap.py --max-zoom 12 --clean
```

Tiles that are entirely ocean or nodata are skipped rather than written, so the
pyramid is much smaller than the tile count would suggest and the sea falls
through to the basemap. Leaflet treats the resulting 404s as empty tiles.

The tiles carry the VALUE, not a colour: each pixel is the score byte, with 0 as
nodata. `index.html` draws them to a canvas and colourizes at draw time, which is
what makes the palette selector, the value-range filter and the exact click
readout possible without regenerating the pyramid. It also means the tiles look
like a grey image if opened directly, which is expected.

`meta.json` carries a `build` id hashed from the pyramid contents, appended to
every tile request. Tile paths are stable across rebuilds, so without it a
browser serves the previous build's tiles and a fix appears to do nothing.

## Deploying to GitHub Pages

Publishing the whole repo would push roughly 1 GB of GeoTIFFs into git, so
publish only this directory.

```bash
cd web
git init -b main
git add .
git commit -m "nature-likeness web map"
git remote add origin git@github.com:<user>/natlik-map.git
git push -u origin main
```

Then in the repo settings, Pages -> Build and deployment -> Deploy from a branch,
branch `main`, folder `/ (root)`.

Check before pushing that `outputs/` and `data/` are not included. They are in the
project `.gitignore`, but this directory is its own repo, so that file does not
apply here.

### Size

GitHub caps a single file at 100 MB and warns above 50 MB. The tiles are all
small PNGs so nothing approaches that. `values.bin` is a few MB. A Pages site is
soft-capped at 1 GB, which the pyramid stays well under at max zoom 12.

Zoom 12 is about 25 m per pixel at this latitude, so it does not resolve the full
10 m detail of the source. Going to zoom 13 roughly quadruples the tile count.
If you want native resolution, host a COG or PMTiles on object storage with range
requests (Cloudflare R2 has a free tier) and point a client at that instead.

## Data notes

The click readout samples `values.bin`, which is coarser than the tiles, so values
are approximate when zoomed in. It is a readout, not an analysis product. For real
work use the GeoTIFFs, where band 1 is the score as `(v - 1) / 254` with byte 0 as
nodata, and band 2 is the ocean mask.

## Licence and attribution

Derived from Google Satellite Embedding V1, ESA WorldCover v200 (2021), JRC Global
Surface Water v1.4 and Copernicus GLO-30. Built on an Earth Engine project
registered as noncommercial, so the outputs are for noncommercial use. Keep the
attribution line in the map footer.
