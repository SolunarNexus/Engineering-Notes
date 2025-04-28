### Store git credentials for easier authentication
When using authentication via HTTP protocol, you need to authenticate with username & password. To avoid typing your username & password repeatedly, store your credentials in cache via git credential manager:
```bash
git config --local credential.helper cache
```
This saves your credentials after authentication and you won't have to type your credentials so often. 
> Note: this is a safer variant than storing your username & password in a plain-text file

