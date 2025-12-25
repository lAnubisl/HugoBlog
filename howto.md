sudo apt update
sudo apt install hugo
hugo new article/title.md

# for local testing
hugo server -D

# for remote testing (GitHub Codespaces)
hugo server --buildDrafts --baseURL "https://glorious-space-umbrella-wrx66qxgqx397w7-1313.app.github.dev/" --appendPort=false