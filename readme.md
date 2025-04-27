Below is a **self‐contained project specification** you can paste into another o4-mini-high prompt. It lays out the overall app and gives a **step-by-step**, **parameterized** description of your spiral + squeeze algorithm so that any developer (or another ChatGPT) can pick it up and implement it.

---

# Project: Priority-Based Masonry Collage with Spiral Packing

## 1. Overview  
A browser-based collage tool where users upload many images, then a custom “spiral & squeeze” layout engine arranges them:

- **Initial placement** in a loose spiral (no overlaps).  
- **Iterative squeezing** toward the center until each image reaches a configurable minimum gutter (e.g. 8 px) from all neighbors.  
- **Priority & resizing** features layered on top for scale-jitter, center affinity, and interactive editing.

## 2. Major Features

1. **Bulk Image Upload**  
   - File input + drag-and-drop.  
   - Configurable upload–time resize: if an image’s natural width or height exceeds **MaxUploadDim**, scale its larger side to a **random** value in [**MinUploadDim**, **MaxUploadDim**], preserving aspect ratio.

2. **Spiral & Squeeze Layout**  
   - **Phase 1: Spiral Placement**  
     - Compute **i** = diagonal of the largest upload-scaled image:  
       ```
       i = √(max(natW,natH)² + min(natW,natH)²)
       ```  
     - Place images one by one in an outward spiral: right, down, left, up, increasing step counts like an Ulam spiral.  
     - No overlap guaranteed because each step moves by at least **i**.

   - **Phase 2: Iterative Squeeze**  
     - Define **k** = minimum gutter (e.g. 8 px) and **j** = squeeze step (1 px).  
     - Repeat until no image moves:  
       1. Iterate images **in reverse spiral order** (outermost first).  
       2. For each image `img`:  
          - **Attempt X-move** toward center by `dx = sign(centerX − img.x) * j`.  
            - If moving `dx` would cause any other image’s bounding box to come closer than **k** in the X-axis, **cancel** the X-move.  
          - **Attempt Y-move** similarly by `dy = sign(centerY − img.y) * j`, canceling if it would break the **k** gutter in Y.
     - End when **no image** can move in either axis.

3. **Priority & Scale**  
   - After squeeze completes, compute each image’s **distance** to center:  
     ```
     d = √((imgCenterX − centerX)² + (imgCenterY − centerY)²)
     ```  
   - Assign a **priority** on a 1–10 scale inversely proportional to **d** (closest → highest).  
   - Users can **override** priority and scale via an edit modal, which triggers a full re-layout.

4. **Interactive Controls**  
   - **Edit Modal** on image click:  
     - Sliders for **scale** (0.5×–2×) and **priority** (1–10).  
     - Arrow buttons to **swap** priority with the nearest neighbor in any direction.  
     - Live display of each image’s current gutter distances (top/left/bottom/right).

5. **Zoom & Pan**  
   - Mouse wheel zoom (20%–300%).  
   - Click-and-drag panning.  
   - Auto-fit after each layout (never upscale above 100%, scrollbars only if manually zoomed beyond fit).

6. **Export**  
   - Buttons to download as **PNG**, **JPG**, or **PDF**.  
   - Configurable **export-scale** (50%, 100%, 150%).  
   - Use `html2canvas` + `jsPDF`.

## 3. Data Structures

- **ImageData** object:
  ```ts
  interface ImageData {
    img: HTMLImageElement;
    // upload-scaled dimensions (locked before layout)
    natW: number;  
    natH: number;
    // jitter scale (from “large” threshold)
    scale: number;
    priority: number;
    // computed during layout
    x: number; y: number; w: number; h: number;
    // neighbor graph (for optional features)
    neighbors: ImageData[];
  }
  ```
- **Global State**:
  ```ts
  imagesData: ImageData[];
  centerX, centerY: number;
  gutterK: number;        // e.g. 8
  squeezeStepJ: number;   // e.g. 1
  spiralStepI: number;    // computed per largest image
  ```

## 4. Detailed Spiral Algorithm

1. **Compute Spiral Step**  
   ```js
   const diagonals = imagesData.map(d=>Math.hypot(d.natW*d.scale, d.natH*d.scale));
   spiralStepI = Math.max(...diagonals);
   ```

2. **Initial Spiral Placement**  
   ```js
   let x = centerX, y = centerY;
   let dir = 0; // 0:right,1:down,2:left,3:up
   let steps = 1;
   for (let idx=0; idx<imagesData.length; idx++) {
     const d = imagesData[idx];
     d.x = x; d.y = y;
     // advance x,y by spiralStepI in current dir, steps times
     for (let rep=0; rep<2; rep++) {
       for (let s=0; s<steps; s++) {
         if (dir===0) x += spiralStepI;
         if (dir===1) y += spiralStepI;
         if (dir===2) x -= spiralStepI;
         if (dir===3) y -= spiralStepI;
         idx++;
         if (idx<imagesData.length) {
           const dd = imagesData[idx];
           dd.x = x; dd.y = y;
         }
       }
       dir = (dir+1) % 4;
     }
     steps++;
   }
   ```

3. **Iterative Squeeze Loop**  
   ```js
   let moved=true;
   while(moved){
     moved = false;
     // outermost-first: reverse the spiral index array
     for (const d of imagesData.slice().reverse()) {
       // move in X
       const dx = Math.sign(centerX - (d.x + d.w/2)) * squeezeStepJ;
       if (canMove(d, dx, 0)) { d.x += dx; moved = true; }
       // move in Y
       const dy = Math.sign(centerY - (d.y + d.h/2)) * squeezeStepJ;
       if (canMove(d, 0, dy)) { d.y += dy; moved = true; }
     }
   }
   ```

4. **Collision/Gutter Check**  
   ```js
   function canMove(d, dx, dy) {
     const r1 = { x:d.x+dx, y:d.y+dy, w:d.w, h:d.h };
     return imagesData.every(o=>{
       if(o===d) return true;
       // require at least gutterK between r1 and o
       return !rectsTooClose(r1, {x:o.x,y:o.y,w:o.w,h:o.h}, gutterK);
     });
   }
   ```

## 5. Configuration Parameters

| Name               | Default | Description                                            |
|--------------------|---------|--------------------------------------------------------|
| MinUploadDim       | 200 px  | Lower bound for random down‐scale at upload            |
| MaxUploadDim       | 800 px  | Upper threshold for random down‐scale at upload        |
| MinScale           | 50 %    | Lower bound of “large‐image” jitter scale              |
| MaxScale           | 150 %   | Upper bound of “large‐image” jitter scale              |
| MinColumnWidth     | 200 px  | Responsive column width for fallback grid layouts      |
| GutterK            | 8 px    | Minimum required space between any two images          |
| SqueezeStepJ       | 1 px    | Pixel increment for each squeeze pass                  |
| ExportScaleOptions | 50/100/150 % | Scales to apply when exporting PNG/JPG/PDF     |

---

### Next Steps for Implementation  
- Wire this spec into your HTML/JS skeleton.  
- Validate with automated tests for small grids (e.g. 4×4 images).  
- Tune performance (e.g. early-exit squeeze when no moves occur).  
- Expose UI settings exactly as outlined for full configurability.

Feel free to copy-paste this spec into your next o4-mini-high prompt for a complete implementation!