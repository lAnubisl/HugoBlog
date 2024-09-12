sudo apt update
sudo apt install hugo
hugo new article/title.md

# for local testing
hugo server -D

# for remote testing (GitHub Codespaces)
hugo server --buildDrafts --baseURL "https://organic-rotary-phone-pj9vv59pwp39pjx-1313.app.github.dev/" --appendPort=false
