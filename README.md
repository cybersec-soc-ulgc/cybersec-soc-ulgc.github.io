# cybersec_soc_ulgc CTF Team

This repository serves the official web page for the `cybersec_soc_ulgc` CTF team.

Join the team at [ULGC Discord Server](https://my.london.ac.uk/group/student/community)!

## Rules for Contributing:

1. Contributors to this repo must be members of either the CTF Team or the Cybersecurity society at ULGC. 

2. If you would like to be added as a contributor, please message the Presider or the Vice-Presider with your ULGC Discord username. **Requests without a Discord username will be ignored.**

## How to Contribute (for developers):

If you have experience with programming, you can contribute in the following ways:

### Write a Blog Post/Challenge Walkthrough

Refer to the [`example.md`](/example.md) file for writing Markdown content.

This repository is arranged with the following directory structure:

```
/content
  |____ /blog
        |____ _index.md
  |____ /members
        |____ index.md
  |____ /writeups
        |____ /ictf-2025-pwn-writeup-babybof
              |____ index.md
  |____ _index.md
```

To create a new blog post, create a new directory inside `/blog` with the blog title. Write your content in an `index.md` file inside this new directory. For example:

```
/content
  |____ /blog
        |____ _index.md
        |____ /new-blog-post <--- new directory
              |____ index.md <--- write your content here
  |____ /members
        |____ index.md
  |____ /writeups
        |____ /ictf-2025-pwn-writeup-babybof
              |____ index.md
  |____ _index.md
```

Follow this same process to create a new Writeup. For adding images, create another directory inside the newly created directory titled `images`. Insert your image files here and reference them in the `index.md` as `images/my-image.png`. For example:

```
/content
  |____ /blog
        |____ _index.md
        |____ /new-blog-post 
              |____ /images <--- new images directory
                    |____ my-image.png <--- new image file
              |____ index.md 
  |____ /members
        |____ index.md
  |____ /writeups
        |____ /ictf-2025-pwn-writeup-babybof
              |____ index.md
  |____ _index.md
```

### Design a new feature

Refer to the [Hugo Documentation](https://gohugo.io/configuration/) for designing new features. Make sure to test the feature locally. Explain the feature in-depth while creating a Pull Request.

## How to Contribute (for non-techs):

If you do not have experience in programming but would like to write a blog post for the Team Page, message the Presiders with a short summary of your blog. We will publish it for you!
