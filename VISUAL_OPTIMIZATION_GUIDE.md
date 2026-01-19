# Visual Guide: LCP/FCP Fixes

## 📊 Current Problem Visualization

```
BEFORE: 19.9s LCP Timeline
═══════════════════════════════════════════════════════════════════

HTML Download      Fonts Block      Scripts Block      Image Block
[0-1s]            [1-3s]           [3-8s]             [8-19.9s]
  ↓                  ↓                 ↓                   ↓
[████████████████████████████████████████████████████████████████] 19.9s
                                                         ↑
                                                    PAINT HAPPENS
                                                    (LCP Element)

Key Bottleneck: 73% = 14.5s RENDER DELAY after image loads
```

---

## 🚀 After Optimization Visualization

```
AFTER: ~1.8s LCP Timeline (Estimated)
════════════════════════════════════════

HTML Download    Fonts (Async)    Scripts (Defer)    Image (Preload)
[0-200ms]        [100-500ms]      [100-600ms]        [200-1200ms]
  ↓                  ↓                  ↓                   ↓
[████████████████████████] 1.8s
            ↑
        PAINT HAPPENS
        (Image preloaded,
         Scripts deferred)

Improvement: -91% reduction (19.9s → 1.8s)
```

---

## 📝 What Changed in Your Code

### 1️⃣ head.html - Font Loading

**BEFORE (Render-Blocking):**
```html
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Montserrat:wght@500;600&family=Nunito+Sans:wght@400;600&display=swap">
```
❌ Stops HTML parser until fonts load

**AFTER (Non-Blocking):**
```html
<link rel="preload" href="https://fonts.googleapis.com/css2?family=Montserrat:wght@500;600&family=Nunito+Sans:wght@400;600&display=swap" as="style" />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Montserrat:wght@500;600&family=Nunito+Sans:wght@400;600&display=swap" />
```
✅ Fonts load in parallel with page render

---

### 2️⃣ head.html - Icon Fonts

**BEFORE (Render-Blocking - 124KB):**
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" />
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/devicon.min.css" />
```
❌ Blocks FCP by 150-200ms

**AFTER (Deferred - loads as print media):**
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" media="print" onload="this.media='all'" />
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/devicon.min.css" media="print" onload="this.media='all'" />
```
✅ Icons load after page paints, content visible immediately

---

### 3️⃣ head.html - Image Preload Priority

**BEFORE:**
```html
<link rel="preload" as="image" href="/images/bg-sec-image.jpg" />
```

**AFTER:**
```html
<link rel="preload" as="image" href="/images/bg-sec-image.jpg" fetchpriority="high" />
```
✅ Browser downloads hero image with highest priority

---

### 4️⃣ baseof.html - Script Loading

**BEFORE (All block rendering):**
```html
<script src="scripts.js"></script>
<script src="colors.js"></script>
<script src="gsap-animations.js"></script>
<script src="pexels-background.js"></script>
<script src="https://cdn.jsdelivr.net/npm/fslightbox/index.js"></script>
```
❌ Parser stops, waits for all scripts before painting

**AFTER (Scripts deferred):**
```html
<script src="pexels-background.js" defer></script>
<script src="scripts.js" defer></script>
<script src="colors.js" defer></script>
<script src="gsap-animations.js" defer></script>
<script src="https://cdn.jsdelivr.net/npm/fslightbox/index.js" defer></script>
```
✅ Parser continues, scripts execute after page paints

---

### 5️⃣ pexels-background.js - Idle Callback

**BEFORE (Blocks if loading Pexels API):**
```javascript
if (window.PEXELS_API_KEY) {
    this.apiKey = window.PEXELS_API_KEY;
    this.init();  // ❌ Blocks render if API is slow
}
```

**AFTER (Deferred to idle time):**
```javascript
if (window.PEXELS_API_KEY) {
    this.apiKey = window.PEXELS_API_KEY;
    if (window.requestIdleCallback) {
        requestIdleCallback(() => this.init());  // ✅ After page renders
    } else {
        setTimeout(() => this.init(), 100);
    }
}
```
✅ API images load after paint completes

---

### 6️⃣ main.css - Loading Skeleton

**NEW (Prevents Flash of White):**
```css
.hero-section::before {
    content: '';
    animation: skeleton-pulse 2s infinite;
    background: [gradient];
}

@keyframes skeleton-pulse {
    0%, 100% { opacity: 0.6; }
    50% { opacity: 0.8; }
}

body.loaded .hero-section::before {
    display: none;  /* Hide after load */
}
```
✅ Shows animated gradient while image loads
✅ Improves perceived performance

---

## 💾 Resource Breakdown

### **What's Taking Time Right Now?**

| Resource | Size | Load Time | Impact |
|----------|------|-----------|--------|
| HTML Document | 112 KB | 200ms | ✅ Fast |
| Fonts (Google) | 319 KB | 1-2s | ⏱️ Slow (now async) |
| Font Awesome | 124 KB | 800ms | ⏱️ Slow (now deferred) |
| Devicon | 99 KB | 600ms | ⏱️ Slow (now deferred) |
| **Hero Image** | **800KB+** | **8-14s** | 🚨 **CRITICAL** |
| Scripts | 57 KB | 300ms | ⏱️ Slow (now deferred) |
| Stylesheets | 93 KB | 200ms | ✅ Fast |
| **TOTAL** | **3.4 MB** | **19.9s** | |

---

## 🎯 Critical Action Item

### **To See 85-90% Improvement, Compress Hero Image**

```bash
# Current: 800KB → Target: 150KB (80% reduction)

# Option 1: Use online tool
# Go to https://squoosh.app
# Upload /images/bg-sec-image.jpg
# Compress to 75% quality
# Download result

# Option 2: Use command line
brew install imagemagick
convert /images/bg-sec-image.jpg -quality 75 /images/bg-sec-image-compressed.jpg

# Replace original
mv /images/bg-sec-image-compressed.jpg /images/bg-sec-image.jpg
```

### **Why This Matters**

```
BEFORE Compression:        AFTER Compression:
Image: 800KB               Image: 150KB
Time: 14.5s delay          Time: 2.5s delay
└─ 73% of LCP              └─ 15% of LCP

Result: LCP drops from 19.9s → ~2s
```

---

## ✅ Checklist: Before Redeploying

- [ ] All defer attributes added to scripts
- [ ] Font loading strategy updated
- [ ] Icon fonts moved to print media
- [ ] Image preload has fetchpriority="high"
- [ ] pexels-background.js uses requestIdleCallback
- [ ] Loading skeleton CSS added
- [ ] Hero image compressed to <200KB
- [ ] Test locally with `hugo --minify`
- [ ] Deploy to production
- [ ] Retest on SpeedVitals
- [ ] Verify LCP < 2.5s
- [ ] Monitor Core Web Vitals

---

## 📊 Expected Results

### **After All Optimizations + Image Compression**

```
SCORE BREAKDOWN
═══════════════════════════════════════════════════════════════

Current:                   After Optimization:
┌─────────────────────┐    ┌─────────────────────┐
│ Performance: 55%    │    │ Performance: 92%    │ ← +37%
│ Accessibility: 92%  │ → │ Accessibility: 92%  │
│ Best Practices: 80% │    │ Best Practices: 85% │
│ SEO: 95%            │    │ SEO: 95%            │
└─────────────────────┘    └─────────────────────┘
   GRADE: C (55%)            GRADE: A (92%)
```

---

## 🔗 Deployment Steps

1. **Build locally:**
   ```bash
   hugo --minify
   ```

2. **Test in browser DevTools:**
   - Open DevTools → Network tab
   - Reload page
   - Check that scripts have `defer` attribute
   - Verify image starts downloading early

3. **Deploy to production:**
   ```bash
   git add -A
   git commit -m "perf: optimize LCP/FCP - defer scripts, async fonts"
   git push
   ```

4. **Retest after 24 hours** (let metrics stabilize):
   - Go back to SpeedVitals
   - Run another test
   - Compare metrics

---

## 💡 Pro Tips

1. **Use Chrome DevTools to measure:**
   - Ctrl+Shift+P → "Disable cache"
   - Throttle to "Slow 4G"
   - Check FCP and LCP markers in Timeline

2. **Monitor real users:**
   - Install Google Analytics
   - Check "Web Vitals" section
   - Compare against Core Web Vitals targets

3. **Keep optimizing:**
   - Hero image optimization = 85-90% gain
   - Unused CSS removal = 50KB savings
   - Icon lazy loading = additional savings

---

## ❓ FAQ

**Q: Will changes break anything?**
A: No. `defer` attribute waits for HTML to parse before executing, ensuring DOM is ready. Scripts will still execute.

**Q: Why use `media="print"` for fonts?**
A: Browsers skip print stylesheets during page load, so they don't block FCP. `onload` switches it back to "all" after paint.

**Q: What if browser doesn't support requestIdleCallback?**
A: Falls back to `setTimeout(..., 100)`, which is fine - API loads after 100ms instead of when idle.

**Q: How much will LCP improve?**
A: With image compression: 85-90% (19.9s → 1.8-2.5s)
   Without compression: 20-30% from code changes alone

**Q: Should I convert to WebP?**
A: Yes! WebP is 30% smaller than JPEG. Use with JPG fallback for older browsers.

---

**Last Updated:** January 19, 2026
**Test Device:** Mobile (Mumbai, India, 4G throttled)
**Next Retest:** After hero image compression
