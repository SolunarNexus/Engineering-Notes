### Store git credentials for easier authentication
When using authentication via HTTP protocol, you need to authenticate with username & password. To avoid typing your username & password repeatedly, store your credentials in cache via git credential manager:
```bash
git config --local credential.helper cache
```
This saves your credentials after authentication and you won't have to type your credentials so often. 
> Note: this is a safer variant than storing your username & password in a plain-text file

### Ignore some files locally (without .gitignore)
Patterns which are specific to a particular repository but which do not need to be shared with other related repositories (e.g., auxiliary files that live inside the repository but are specific to one user's workflow) should go into the `$GIT_DIR/info/exclude` file. This file has the same format as any `.gitignore` file.

Note, if you already have unstaged changes you must run the following after editing your ignore-patterns:
```sh
git update-index --assume-unchanged <file-list>
```
