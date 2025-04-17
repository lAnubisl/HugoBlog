sudo apt update
sudo apt install hugo
hugo new article/title.md

# for local testing
hugo server -D

# for remote testing (GitHub Codespaces)
hugo server --buildDrafts --baseURL "https://ubiquitous-yodel-69w66xwr4jc4jj7-1313.app.github.dev/" --appendPort=false