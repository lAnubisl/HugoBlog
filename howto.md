sudo apt update
sudo apt install hugo
hugo new article/title.md

# for local testing
hugo server -D

# for remote testing (GitHub Codespaces)
hugo server --buildDrafts --baseURL "https://bug-free-journey-x5x66gxwr42vxqw-1313.app.github.dev/" --appendPort=false
