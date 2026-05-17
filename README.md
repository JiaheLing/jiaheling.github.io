# jiaheling-site

A starter personal website for Cloudflare Pages Direct Upload. This version uses a smaller left-side profile image, smaller typography, and a soft off-white background.

## Files

- `index.html` — homepage
- `assets/style.css` — shared styling
- `pages/education.html`
- `pages/research.html`
- `pages/projects.html`
- `pages/photolab.html`

## Upload to Cloudflare Pages

Upload the contents of this folder, or zip the folder and upload it through Cloudflare Pages Direct Upload.

## Customize

1. Replace placeholder biography text in `index.html`.
2. Update email, GitHub, Scholar, and CV links.
3. Replace the portrait placeholder with your own image:
   - add your photo as `assets/profile.jpg`
   - replace the `portrait-placeholder` block in `index.html` with:

```html
<img class="portrait-img" src="assets/profile.jpg" alt="Portrait of Jiahe Ling">
```

Then add this CSS:

```css
.portrait-img {
  width: 100%;
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  display: block;
}
```
