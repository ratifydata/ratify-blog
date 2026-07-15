# ratify-blog

Source for the Ratify blog, published via GitHub Pages at:

https://ratifydata.github.io/ratify-blog

## Adding a post

1. Create a new file in `_posts/` named `YYYY-MM-DD-title.md`
   (e.g. `2026-07-20-why-we-built-ratify.md`).
2. Start the file with this frontmatter block:

   ```yaml
   ---
   layout: post
   title: "Your Post Title"
   date: 2026-07-20
   ---
   ```

3. Write the post in Markdown below the frontmatter.
4. Commit and push to `main`. GitHub Pages rebuilds automatically,
   usually live within a minute or two.

No local build step is required, GitHub builds the Jekyll site for you.
