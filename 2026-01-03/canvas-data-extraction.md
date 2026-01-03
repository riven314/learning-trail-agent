# Canvas Data Extraction

Context: learned techniques for extracting time series data from canvas-rendered charts when traditional DOM querying fails. Session directory: not available.

## Core Mental Model: Rendering Method Determines Data Accessibility

**SVG/DOM Rendering**: data persists as queryable DOM elements
```html
<svg>
  <circle cx="150" cy="200" data-value="1523.45" />
</svg>
<!-- query anytime with document.querySelectorAll() -->
```

**Canvas Rendering**: data is consumed and discarded after rendering
```javascript
ctx.lineTo(150, 200);  // coordinate lost after pixels painted
ctx.stroke();
```

## What Canvas Actually Stores

Canvas is a **bitmap buffer**, not a vector store. It holds a 2D array of RGBA pixel values in memory.

```
Memory: Uint8ClampedArray [R, G, B, A, R, G, B, A, ...]
For 400×200 canvas: 400 × 200 × 4 = 320,000 bytes
```

What you can extract:
```javascript
const imageData = ctx.getImageData(0, 0, width, height);
// Returns only: { data: Uint8ClampedArray, width, height }
// No coordinates. No semantic meaning. Just pixel colors.
```

Canvas vs PNG/JPG:
- Canvas: raw RGBA in memory, modifiable, no compression
- PNG/JPG: compressed file on disk, immutable without decode/re-encode
- Relationship: canvas exports to PNG via `canvas.toDataURL()`

## Data Lifecycle: The Critical Hook Point

```
Data Fetch → Process in JS/React → Render to Canvas → Garbage Collected
                                         ↑
                                    HOOK HERE
                                   (last chance!)
```

Data exists only transiently during the draw call. After rendering completes, coordinates are permanently lost.

## Pattern: Intercepting Draw Calls

Override canvas methods to capture data before it's discarded:

```javascript
const capturedData = [];
const originalLineTo = CanvasRenderingContext2D.prototype.lineTo;

CanvasRenderingContext2D.prototype.lineTo = function(x, y) {
    capturedData.push({ x, y, timestamp: Date.now() });
    return originalLineTo.call(this, x, y);  // preserve rendering
};
```

**Critical requirement**: hook must be installed BEFORE chart renders.

## DevTools Injection Techniques

**Breakpoint Injection** (most reliable):
1. DevTools → Sources → Event Listener Breakpoints
2. Enable "Script" → "Script First Statement"
3. Refresh page (pauses immediately)
4. Paste hook code in Console
5. Resume execution (F8)

**Snippets + Refresh**:
1. DevTools → Sources → Snippets
2. Create snippet with hook code
3. Run snippet → Refresh page

**Local Overrides**:
1. DevTools → Sources → Overrides
2. Set override folder
3. Modify site's JS to include hook

## Alternative Approaches

When hooking isn't feasible:
- **Network interception** (mitmproxy, Charles): catch data at source
- **React DevTools**: access component state directly
- **Official API**: cleanest solution if available
- **Exchange API**: query data source directly
- **Computer vision**: last resort, lossy and error-prone

## Decision Tree

```
SVG chart? → Query DOM directly
Canvas chart?
  → API exists? → Use API
  → Intercept network? → Use proxy/fetch hooks
  → Hook before render? → Override canvas methods
  → Otherwise → Computer vision or manual logging
```

#WEB-SCRAPING #CANVAS #JAVASCRIPT #DATA-EXTRACTION
