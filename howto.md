sudo apt update
sudo apt install hugo
hugo new article/title.md

# for local testing
hugo server -D

# for remote testing (GitHub Codespaces)
hugo server --buildDrafts --baseURL "https://improved-space-enigma-pj9vv59r6725q-1313.app.github.dev/" --appendPort=false
