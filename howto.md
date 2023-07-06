sudo apt update
sudo apt install hugo
hugo new article/title.md

# for local testing
hugo server -D

# for remote testing (GitHub Codespaces)
hugo server --buildDrafts --baseURL "https://lanubisl-fantastic-computing-machine-v6w44jwg7qh56p-1313.preview.app.github.dev" --appendPort=false
