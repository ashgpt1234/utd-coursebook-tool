
# What is This?

This tool will:

1. Log in to CourseBook.
1. Download XLSX rosters.
1. Download PDF photorosters.
1. If there is more than one section for a course then a CSV file with NetID and section (useful to upload to Blackboard and make a "course section" column).
1. Create an image file for every student.

# How to Use

1. Get a copy by doing one of:
    1. Download this [zip file](https://github.com/bmccary/utd-coursebook-tool/archive/master.zip).
    1. Clone this repository: `git clone https://github.com/bmccary/utd-coursebook-tool.git`
1. In the `utd-coursebook-tool` directory, copy the file `config.py.example` to `config.py` and edit it (with a text editor) to correspond to your situation.
1. At a terminal in the `utd-coursebook-tool` directory, type: `scons`
    1. You'll be prompted for your UTD password. A web browser will be started. 
    1. Assuming you typed your password correctly, you can watch it log in and download all of your files from CourseBook.
    1. If you didn't log in correctly then the process will time out after about 10 seconds. You just run `scons` again.
1. And one more time in the terminal, type: `scons`
    1. You have to do this one more time because `scons` only decides *at the beginning* what actions need to be taken, and these decisions might change if new files are downloaded.

Notably, this process is inherently serial (because it is driving a web browser) so don't try to run `scons` in parallel mode.

# Prerequisite Software

You'll need python 2.7+, and a few python packages. Once you have python installed, at a terminal:

```
pip install --user openpyxl
pip install --user scons
pip install --user selenium
```

