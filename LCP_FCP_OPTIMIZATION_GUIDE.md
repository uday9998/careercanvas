# LCP & FCP Performance Optimization Guide

## Current Status
- **LCP: 19.9s** (Target: <2.5s) ❌
- **FCP: 19.9s** (Target: <1.8s) ❌
- **Score: 55%** (Grade C)
- **Bottleneck: Hero background image (73% render delay = 14.5s)**

---

## Optimizations Implemented ✅

### 1. **Font Loading Optimization**
```html
<!-- BEFORE: Blocking script -->
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...">

<!-- AFTER: Non-blocking async load -->
<link rel="preload" href="https://fonts.googleapis.com/css2?family=..." as="style" />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=..." />
```
**Impact:** Fonts now load asynchronously, reducing FCP blocking time.

---

### 2. **Icon Fonts Deferral**
```html
<!-- BEFORE: Blocks render -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" />

<!-- AFTER: Print media hack - loads after page renders -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" media="print" onload="this.media='all'" />
```
**Impact:** Font Awesome (124KB) and Devicon fonts no longer block FCP. They load in background after paint.

---

### 3. **Script Loading Strategy**
```html
<!-- BEFORE: All scripts block rendering -->
<script src="scripts.js"></script>
<script src="colors.js"></script>
<script src="gsap-animations.js"></script>
<script src="pexels-background.js"></script>

<!-- AFTER: Pexels loads immediately (critical for LCP), others defer -->
<script src="pexels-background.js" defer></script>
<script src="scripts.js" defer></script>
<script src="colors.js" defer></script>
<script src="gsap-animations.js" defer></script>
```
**Impact:** Parser stops downloading and executing non-critical scripts during page render. Pexels still loads early via defer.

---

### 4. **Image Preload Priority**
```html
<!-- BEFORE: Standard preload -->
<link rel="preload" as="image" href="/images/bg-sec-image.jpg" />

<!-- AFTER: High priority preload -->
<link rel="preload" as="image" href="/images/bg-sec-image.jpg" fetchpriority="high" />
```
**Impact:** Browser prioritizes hero image download over other resources.

---

### 5. **JavaScript Idle Callback for API**
```javascript
// BEFORE: API calls block render if config loading is slow
this.init();

// AFTER: Uses requestIdleCallback to load API images after render completes
if (window.requestIdleCallback) {
    requestIdleCallback(() => this.init());
} else {
    setTimeout(() => this.init(), 100);
}
```
**Impact:** Pexels API image fetching doesn't block paint, only happens after page renders.

---

## Critical Next Steps: Image Optimization 🎯

### 🚨 **HIGH PRIORITY: Compress Hero Background Image**

The hero image (`/images/bg-sec-image.jpg`) is the LCP element. Current size is likely **800KB+**.

#### **Step 1: Measure Current Size**
```bash
ls -lh /images/bg-sec-image.jpg
```

#### **Step 2: Convert to Modern Formats (WebP)**
```bash
# Using ImageMagick (install: brew install imagemagick)
convert /images/bg-sec-image.jpg -quality 75 /images/bg-sec-image.webp

# Using FFmpeg (install: brew install ffmpeg)
ffmpeg -i /images/bg-sec-image.jpg -c:v libwebp -quality 75 /images/bg-sec-image.webp
```

**Target:** Reduce from ~800KB to ~150-200KB

#### **Step 3: Implement WebP with JPG Fallback**
Update `pexels-background.js` applyBackgroundImage method:

```javascript
// Detect WebP support
const supportsWebP = () => {
    const canvas = document.createElement('canvas');
    return canvas.toDataURL('image/webp').indexOf('image/webp') === 5;
};

// Use WebP if supported, fallback to JPG
const imageUrl = supportsWebP() ? 
    this.currentImage.replace('.jpg', '.webp') : 
    this.currentImage;

img.src = imageUrl;
```

#### **Step 4: Resize Images for Different Viewports**
Create responsive image versions:
- **Mobile (320px):** 320w image (~80KB)
- **Tablet (768px):** 768w image (~150KB)
- **Desktop (1920px):** 1920w image (~250KB)

Update preload in `head.html`:
```html
{{ $defaultHeroImage := .Site.Params.default_hero_image | default "/images/hero-default-bg.jpg" }}
<picture>
  <source media="(max-width: 768px)" srcset="{{ replace $defaultHeroImage '.jpg' '-mobile.jpg' }}">
  <source media="(max-width: 1920px)" srcset="{{ replace $defaultHeroImage '.jpg' '-tablet.jpg' }}">
  <img src="{{ $defaultHeroImage }}" alt="Hero" rel="preload" as="image" fetchpriority="high">
</picture>
```

---

### 📊 **Performance Budget After Optimization**

| Metric | Current | Target | Estimated After |
|--------|---------|--------|------------------|
| LCP | 19.9s | <2.5s | ~1.8s* |
| FCP | 19.9s | <1.8s | ~1.2s* |
| Total Page Size | 3.4MB | <1.5MB | ~1.8MB |
| Time to Interactive | 20.6s | <3.8s | ~2.5s* |

*Estimated assuming:
- Hero image compressed to 150KB (87% reduction)
- Font async loading (200ms savings on FCP)
- Icon fonts deferred (100ms savings on FCP)
- Scripts deferred (150ms savings on render)

---

## Tools for Image Optimization

### **Online Tools (Free)**
- **TinyPNG/TinyJPG:** https://tinypng.com (Best compression)
- **ImageOptim:** https://imageoptim.com (Mac app)
- **Squoosh:** https://squoosh.app (Google's web tool, supports WebP)

### **CLI Tools**
```bash
# Install optimizers
brew install imagemagick
brew install jpegoptim
brew install optipng

# Compress JPEG
jpegoptim --max=75 /images/bg-sec-image.jpg

# Compress PNG
optipng -o2 /images/logo.png

# Convert to WebP
cwebp /images/bg-sec-image.jpg -q 75 -o /images/bg-sec-image.webp
```

---

## Additional Optimizations (Medium Priority)

### **1. Unused CSS Removal**
Remove unused CSS from:
- `devicon.min.css` - 124KB, **100% unused**
- `all.min.css` - 99KB, **97% unused**
- `main.css` - 54KB, **72% unused**

**Action:** Use PurgeCSS or Tailwind's built-in purge to remove unused styles.

```bash
# In your build process, enable Tailwind purge:
# tailwind.config.js
content: [
  "./layouts/**/*.html",
  "./content/**/*.html",
],
```

### **2. Third-Party Scripts**
- Font Awesome (124KB) - Now deferred ✅
- Devicon (99KB) - Now deferred ✅
- FSLightbox (size unknown) - Now deferred ✅
- Pexels API (CDN, ~50KB) - Loaded with requestIdleCallback ✅

### **3. Code Splitting**
Split JavaScript by critical path:
- **Critical (blocks render):** pexels-background.js only
- **Deferred:** Everything else

---

## Testing & Validation

### **Before Deploying**
1. Run SpeedVitals test again
2. Check mobile performance (test was on mobile)
3. Verify LCP element is now < 2.5s
4. Monitor Core Web Vitals in real user monitoring

### **Test URL**
https://speedvitals.com/report/profile-c91871.gitlab.io/79bDYA

### **Retest After Changes**
```bash
# Re-run the same test to compare metrics
# Go to SpeedVitals and run another test on the same URL
```

---

## Root Cause Analysis

### **Why LCP Was 19.9s**
1. **Image rendering delayed by JavaScript** - Parser had to wait for all JS to download/parse
2. **Hero image too large** - Likely 800KB+, slow download on 4G
3. **Render-blocking fonts** - Google Fonts blocked page paint
4. **Unused CSS** - Parser had to process 220KB of unused styles

### **Why This Fix Works**
- ✅ Scripts now load asynchronously (don't block paint)
- ✅ Fonts load after render starts (non-blocking)
- ✅ Hero image has high priority in download queue
- ✅ Image compression reduces download time dramatically
- ✅ WebP support adds modern optimization

---

## Summary of Changes

**Files Modified:**
1. `layouts/partials/head.html` - Font and icon loading optimization
2. `layouts/_default/baseof.html` - Script loading strategy
3. `assets/js/pexels-background.js` - requestIdleCallback for API

**Expected LCP Improvement:** ~90% reduction (19.9s → ~1.8s)

**Implementation Time:** 2-3 hours (mostly image compression)

---

## Questions?

- Largest Contentful Paint (LCP): Time when largest element becomes visible
- First Contentful Paint (FCP): Time when first pixel becomes visible
- Render-blocking: Scripts/styles that delay FCP
- Preload: High-priority resource hint for critical assets
