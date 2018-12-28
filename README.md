# Generating favicons from SVG

```bash
npm install icon-gen -g
npm install svgexport -g

icon-gen -i static/favicon.svg -o static/ --ico name=favicon sizes=16,24,32,48,64

svgexport static/apple-touch-icon.svg static/apple-touch-icon.png 1024:1024
for size in 120 152 167 180; do
	svgexport static/apple-touch-icon.svg static/apple-touch-icon-${size}x${size}.png $size:$size
done
```
