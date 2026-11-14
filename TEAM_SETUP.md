# Team setup: Earth Analogs Explorer

## A. Just test the website (everyone)
Open the GitHub Pages link: PAGES_LINK
That is all. Hard-reload (Ctrl+Shift+R) after Thant says new data is pushed.

To run it on your own computer instead:
1. Install Git and Python 3.11 or newer.
2. Open a terminal (Windows: Command Prompt) and run:
   git clone REPO_LINK
   cd REPO_NAME
   python -m http.server 8123 --directory docs
3. Open http://localhost:8123 in your browser.
4. To get the latest version later: git pull

## B. Run the data pipeline (Please let everyone know before you run the data pipeline)
1. Make a free NASA Earthdata account: https://urs.earthdata.nasa.gov/users/new
   Then in your Earthdata profile, under Applications > Authorized Apps, approve "LP DAAC Data Pool".
2. In the project folder:
   pip install -r pipeline/requirements.txt
   python pipeline/fetch_data.py
   It asks for your Earthdata username and password, then downloads about 2.3 GB.
   If it stops, run the same command again: it continues where it stopped.
   It must end with "all files present".
3. Rules:
   - Never commit anything in pipeline/raw/ (git already ignores it).
   - Never edit files in docs/data by hand: rerun the script that makes them.
   - Only one person edits docs/index.html or commits docs/data at a time. Ask in the group first.
