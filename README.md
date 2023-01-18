```
   ____
  / __/__ ____ _______ __
 _\ \/ _ `/ _ `/ __/ // /
/___/\_,_/\_,_/_/  \_,_/

A Static Site Generator for Fun and Profit
```

# Saaru

Saaru is an opinonated Static Site Generator written in Rust. It uses Markdown and Jinja to render pure HTML/CSS Websites from your markdown + jinja source.

## Usage

Running the program is simple. Once you have the repository cloned, you can use the following command to check oout the example site ->

```bash
$ cargo run --release -- --base_path <your example_source directory>
```

Feel free to base your site off of the `docs` directory, which already has a bunch of templates pre-defined for you. It's got my name in there, but TODO Refactor soon enough.

```bash
$ cargo run --release -- --base_path ./example_source
```

If nothing's wrong, your entire site as HTML and CSS will present itself in the `./docs/build` directory. From then onwards, all you need to do is launch a web server with `./docs/build` as the source such as [this package](https://www.npmjs.com/package/serve).

As of right now, Saaru is a little opinioniated on how exactly you should structure your site. As of right now, it boils down to having a folder with the following structure =>

```

docs/
├── src
│   ├── index.md
│   ├── internals
│   │   ├── collections.md
│   │   ├── deep_data_merge.md
│   │   └── tags.md
│   └── markdown.md
└── templates
    ├── base.jinja
    ├── footer.jinja
    ├── index.jinja
    ├── menubar.jinja
    ├── post_index.jinja
    ├── post.jinja
    ├── post_new.jinja
    ├── tags.jinja
    └── tags_page.jinja

```

It's possible to have an abitrary configuration of files in the `src` folder, so long as you've got each and every markdown document with the right frontmatter.

Here's the absolute minimum frontmatter. ~(This will be iterated on, but as of now - ) There must be **A MINIMUM OF ONE TAG PER POST**.~

```yaml
---
title: <A title for your post>
description: <a description>
template: post.jinja # This must be a valid template from the `templates` directory
tags:
  - <example tag 1>
  - <example tag 2>
collections:
  - <example collection>
  - <example collection 2>
---
```

## Live Reload

As of right now, Live reload is enabled by default, and is hidden behind a command line flag.

```bash
$ cargo run --release -- --base_path ./example_source --live-reload
```

As and when you make a change to a file and save the file, Saaru will re-render that file into the build directory. On your browser (or if your web server supports watching the file system, do nothing - ), hit refresh to see your content updated.

### Etymology

Saaru means Rasam, which is a type of spicy, thin lentil soup, often eaten with rice. This project is called Saaru because I like Saaru very much.

### Nonsense Formal explanation

SAARU -> StAtic Almanac Renderer and Unifier

## TODO

- [ ] Fix the error handling
- [ ] [docs] Specify what the minimum supported file structure for opinionated mode is
- [ ] Parallelized rendering
- [ ] Web Server
- [ ] Delete Build Directory on re-render
- [x] Custom Info JSON File - for defaults, fixed params, etc (Perhaps a `.saaru.json`)
  ```json
  {
    "default_dir": "abc",
    "default_template": "some_template.jinja",
    "author": {
      "name": "Somesh",
      "bio": "this is my bio",
      "twitter": "..."
    }
  }
  ```
- [x] tree-shaken rendering, only re-render what's changed?
- [x] Live reload?
- [x] Run Pre-flight checks (check if templates dir exists, check if source dir exists, etc)
- [x] External CSS / Custom CSS injection
- [x] Refactor!
- [WIP] Make the Saaru Docs a Saaru-generated website
- [x] Make all frontmatter optional (only `title` and `description` are now mandatory)
- [x] Static Directory Support (~~Minify CSS and Build~~ copy over all ~~other~~ static files)

## Data Merge Architecture

Each and every `.md` file has frontmatter, which it uses to determine which collections it's a part of.

```yaml
---
title: something
description: something else
collections:
  - posts
tags:
  - computerscience
  - space
  - alphabet
---
```

The first pass of the SSG Renderer is a frontmatter pass, where inverted indices (think TF-IDF) of both collections and tags are created. These indices are then accessible in frontmatter such that one can easily generate an index page for every document present here.

Thus, our generated indices will look something like this ->

```yaml
collection_map:
  - posts: [something]
tag_map:
  - computerscience: [something]
  - space: [something]
  - alphabet: [something]
```

It is also possible to auto-generate the tags pages such that lookups are possible on the basis of tags.

### The Frontmatter Map

This is the primary index for every file being considered in the static site generator.

### New Render Pipeline

This might need to be rearchitected to include live reload when it happens.

In this architecture, there are two passes that go into rendering files - this is architectured to make the deep data merge possible.

1. Frontmatter Pass -
   - Read every File in the source directory
   - Capture all the frontmatter in structs
   - Read all the Markdown Content and store it (but do not convert it to HTML) - this is so we don't need to read it again to save on I/O
   - Create all the indices/maps on the fly (collection_map, tag_map) at this point
   - Capture structure - `HashMap<Path, AugmentedFrontmatter>` where `AugmentedFrontmatter` has the frontmatter, read content markdown (and possibly later have live reload based on file changes)

> By the end of the frontmatter pass, all collections and collection data must be satisfied, should any other template wish to read it

2. Render Pass
   - Iterate through the entire hashmap, passing the collections available as a part of the context
   - Render the markdown present and write it to file
   - Now the entire set of templates has access to the data acquired in the merge.

## The Frontmatter passed to every file

Every file gets the following frontmatter passed into it ->

```rust
let rendered_final_html = rendered_template
.render(context!(
      frontmatter => input_aug_frontmatter.frontmatter,
      postcontent => html_output,
      tags => &self.tag_map,
      collections => &self.collection_map,
      ))
.unwrap();
```

- `frontmatter` -> The frontmatter for the post
- `postcontent` -> The parsed markdown and HTML for the post
- `tags` -> The tags sitewide, structured as `Vec<tag: String, Vec<Post: String>>`
- `collections` -> The collections sitewide, structured as `Vec<collection: String, Vec<Post: String>>`

Feel free to use any of these in your pages!

## Default Arguments

this is the default JSON used if there's no `.saaru.json` present in the base folder. The field `metadata.templates.default` field is compulsory, or else Saaru will look for a `post.jinja` in your template environment.

```json
{
  "metadata": {
    "author": {
      "name": "Author",
      "one_line_desc": "hello, world!",
      "twitter": "twitter.com/username",
      "github": "github.com/username"
    },
    "templates": {
      "default": "post.jinja"
    }
  }
}
```
