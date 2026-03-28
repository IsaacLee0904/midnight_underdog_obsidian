# Github

tags: #github #git #version-control
source: [[Isaac's Note]]

---

## ipynb to Github

### Create a new repository

Open Github and create a new repository.

### Add file to Github

Add file > Upload files > choose your files > Commit changes

### Deal with plotly present issue

Github performs a static render of the notebooks and it doesn't include the embedded HTML / JavaScript that makes up a plotly graph.

**1. Present the complete ipynb**

Copy permalink of the ipynb on Github, then put the github link to [https://nbviewer.org](https://nbviewer.org/).

**2. Present only one figure**

Add code in ipynb to turn plotly figure to html (local):

```python
# fig to html
fig.show(renderer = 'chrome')
pio.write_html(fig, file = 'index.html', auto_open = True)
```

Save Page As, then publish the HTML file to Github:

Add file > Upload files > choose your files (html) > Commit changes

Then go to Settings > Pages > link to get the published URL.

---

## Clone a Repository from Github

### Create a new folder

```bash
cd Desktop
mkdir github
```

### Change path to new folder

```bash
cd github
```

### Clone github repository to new folder

```bash
git clone [github repos HTTPS]
```
