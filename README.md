# How to run locally

```
cd detectivecrow-blog

bundle install
bundle exec jekyll serve
```

# Jekyll theme

- https://github.com/cotes2020/jekyll-theme-chirpy

# Semantic Commit Messages

See how a minor change to your commit message style can make you a better programmer.

Format: `<type>(<scope>): <subject>`

`<scope>` is optional

## Example

```
add(20221006): test page
^-^^--------^  ^-------^
|  |           |
|  |           +-> Summary in present tense.
|  |
|  +--------> create or updated post.
|
+-------> Type: add, edit, update, test, chore.
```

More Examples:

- `add`: (add new post)
- `edit`: (edit old post)
- `fix`: (fix setting issues)
- `update`: (update page scripts or framework)
- `test`: (adding missing tests, refactoring tests; no production code change)
- `chore`: (updating grunt tasks etc; no production code change)

References:

- https://gist.github.com/joshbuchea/6f47e86d2510bce28f8e7f42ae84c716
- https://www.conventionalcommits.org/
- https://seesparkbox.com/foundry/semantic_commit_messages
- http://karma-runner.github.io/1.0/dev/git-commit-msg.html
