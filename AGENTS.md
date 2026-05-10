
# About new post
- use `hugo new posts/{title}.md` to create a new post, `{title}.md` should be under path: `content/posts`
- preferred title: notes_on_xxx (for reading books), thinking_of_xxx (for topic discussion), using_xxx (for tech notes or manual)
- prefer starting with a very brief intro about why write this post
- prefer reuseing existing tags
- use `[link to another post]({{< ref "posts/post_name.md>}})` to link to related posts, or `[tag link]({{< tagref "tag_name"}})` for tags.
- use `![caption](/{post_name}/{image_name})` to reference images, image file should be under path: `static/{post_name}`. 
- use MathJax grammar for math formular

# About theme customization
- i use a modified papermod theme as a submodule under path: `themes/Papermod`


# About local dev
- run `hugo server -D` to preview in realtime.

# About deployment
- use hugo 0.146.5 and do not upgrade unless be told.
- push to github and will auto run daily build and deploy to gitbook.