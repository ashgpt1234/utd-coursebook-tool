
from __future__ import print_function

Import('env')

import config

for course in config.COURSE:
    course['prefix'] = course['prefix'].lower()

COURSEBOOK_URL = 'https://coursebook.utdallas.edu'
LOGIN_URL = COURSEBOOK_URL + '/login/coursebook'
PDF_URL = COURSEBOOK_URL + '/prosterpdf/{prefix}{number}.{section}.{term}'
XLSX_URL = COURSEBOOK_URL + '/reportmonkey/class_roster/{prefix}{number}.{section}.{term}'

DRIVER = None

from selenium import webdriver
import time
import os
import getpass

def _login():
    global DRIVER

    print('\n'*8)
    print('Preparing to log-in to ', COURSEBOOK_URL)
    print('\n'*8)
    password = getpass.getpass('Password for NetID {}: '.format(config.NETID))

    options = webdriver.ChromeOptions()
    options.add_argument('disable-java')
    options.add_experimental_option('prefs', 
                    {
                        'download.default_directory': 'download',
                        'download.prompt_for_download': False,
                        'download.directory_upgrade': True,
                        'plugins.always_open_pdf_externally': True,
                    }
                ) 
    DRIVER = webdriver.Chrome(chrome_options=options)
    DRIVER.implicitly_wait(30)

    DRIVER.get(LOGIN_URL)

    username_element = DRIVER.find_element_by_id("netid")
    password_element = DRIVER.find_element_by_id("password")
    login_button     = DRIVER.find_element_by_id("login-button")
    
    username_element.clear()
    password_element.clear()

    username_element.send_keys(config.NETID)
    password_element.send_keys(password)
    login_button.click()

from random import random

def _wait(a, b):
    time.sleep(a + b*random())

from glob import glob

def _download_xlsx(target, source, env):
    X, Y = os.path.split(str(target[0]))
    term, prefix, number, section = X.split('-')
    D = dict(term=term, prefix=prefix, number=number, section=section)

    global DRIVER
    if DRIVER is None:
        _login()

    DRIVER.get(COURSEBOOK_URL)
    _wait(3, 2)

    DRIVER.get(XLSX_URL.format(**D))

    pattern = os.path.join('download', 'roster-{prefix}{number}.{section}.{term}-*.xlsx'.format(**D))
    for i in range(20):
        if glob(pattern):
            break
        _wait(1, 1)
    else:
        raise Exception('Failed to download: {}'.format(pattern))
    _wait(1, 1)

    x = sorted(glob(pattern))[-1]

    Execute(Move(target[0], x))

    for x in glob(pattern):
        Execute(Delete(x))

def _download_pdf(target, source, env):
    X, Y = os.path.split(str(target[0]))
    term, prefix, number, section = X.split('-')
    D = dict(term=term, prefix=prefix, number=number, section=section)

    global DRIVER
    if DRIVER is None:
        _login()

    DRIVER.get(COURSEBOOK_URL)
    _wait(3, 2)

    DRIVER.get(PDF_URL.format(**D))
    _wait(3, 2)

    # For reasons unclear to me, this is necessary
    DRIVER.refresh()

    pattern = os.path.join('download', 'utd-roster-for-{prefix}{number}.{section}.{term}.pdf'.format(**D))
    for i in range(20):
        if glob(pattern):
            break
        _wait(1, 1)
    else:
        raise Exception('Failed to download: {}'.format(pattern))
    _wait(1, 1)

    x = sorted(glob(pattern))[-1]

    Execute(Move(target[0], x))

    for x in glob(pattern):
        Execute(Delete(x))



#
# A monkey-patch to get pandas and SCons to work together, because
# pandas and SCons have different, contrary assumptions about pickle
#
# <monkey-patch>
import sys
import imp
del sys.modules['pickle']
del sys.modules['cPickle']
sys.modules['pickle'] = imp.load_module('pickle', *imp.find_module('pickle'))
sys.modules['cPickle'] = imp.load_module('cPickle', *imp.find_module('cPickle'))
# </monkey-patch>


import openpyxl
import csv
from operator import itemgetter

def _xlsx_to_csv(target, source, env):
    
    wb = openpyxl.load_workbook(str(source[0]))
    ws = wb.active

    def g():
        for row in ws.rows:
            yield [x.value for x in row]
    rows = list(g())

    wb.close()

    rows = rows[2:]
    
    fieldnames = [str(x) for x in rows[0]]
    def g():
        for row in rows[1:]:
            if row[0] is None:
                continue
            yield {x: str(y) for x, y in zip(fieldnames, row)}
    rows = sorted(g(), key=itemgetter('Last Name', 'First Name', 'NetId'))

    def g():
        for row in rows:
            if row['Units'] == 'DROP':
                continue
            yield row

    with open(str(target[0]), 'w') as w:
        writer = csv.DictWriter(w, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(g())

PDFIMAGES_PREFIX = 'i'
PDFIMAGES_PATTERN = PDFIMAGES_PREFIX + '-{:03d}.ppm'

def _pdf_to_ppm_emitter(target, source, env):
    while len(target) > 0:
        target.pop()
    csv0 = str(source[1])
    if os.path.exists(csv0):
        def g():
            tail, head = os.path.split(str(source[0]))
            with open(str(source[1]), 'r') as r:
                reader = csv.DictReader(r)
                for row in reader:
                    yield os.path.join(tail, 'image', row['NetId']) + '.ppm'
        target.extend(g())
    return target, source

def _ppm_name_fix(target, source, env):
    for i, x in enumerate(target):
        tail, head = os.path.split(str(x))
        Execute(Move(x, os.path.join(tail, PDFIMAGES_PATTERN.format(i))))

_pdf_to_ppm = Action(['mkdir -p ${TARGET.dir}', 'pdfimages $SOURCE ${TARGET.dir}' + os.path.sep + PDFIMAGES_PREFIX, _ppm_name_fix])

env['BUILDERS']['PDFtoPPM'] = Builder(action=_pdf_to_ppm, emitter=_pdf_to_ppm_emitter)

def _ppm_to_png_generator(target, source, env, for_signature):
    t = str(target[0])
    tail, head = os.path.split(t)
    netid, _ = os.path.splitext(head)
    with open(str(source[1]), 'r') as r:
        reader = csv.DictReader(r)
        for row in reader:
            if row['NetId'] == netid:
                break
        else:
            raise Exception('NetId "{netid}" not found: {source}'.format(netid=netid, source=str(source[1])))
    return r'''convert -undercolor White -gravity South -pointsize 20 -annotate 0 "{lname}, {fname}\n{netid}" $SOURCE $TARGET'''.format(lname=row['Last Name'], fname=row['First Name'], netid=netid)

env['BUILDERS']['PPMtoPNG'] = Builder(generator=_ppm_to_png_generator, suffix='.png', src_suffix='.ppm')

from itertools import groupby

def _section_csv(target, source, env):
    def g():
        for x in source[1:]:
            x = str(x)
            tail, head = os.path.split(x)
            term, prefix, number, section = tail.split('-')
            with open(x, 'r') as r:
                reader = csv.DictReader(r)
                for row in reader:
                    yield {'Username': row['NetId'], config.SECTION_COLUMN: section}
    X = sorted(g(), key=itemgetter('Username', config.SECTION_COLUMN))
    def g():
        for netid, rows in groupby(X, itemgetter('Username')):
            rows = list(rows)
            sections = set(row[config.SECTION_COLUMN] for row in rows)
            if len(sections) > 1:
                raise Exception('Non-unique section for "{netid}": {sections}'.format(netid=netid, sections=', '.join(sorted(sections))))
            yield rows[0]
    fieldnames = ['Username', config.SECTION_COLUMN,]
    with open(str(target[0]), 'w') as w:
        writer = csv.DictWriter(w, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(g())

def section_csv_action(section_column):
    def f(target, source, env):
        return _section_csv(target, source, env, section_column)
    return f

def g():
    for course in config.COURSE:
        for x in course['section']:
            pdf = env.Command(target=os.path.join('{term}-{prefix}-{number}-{x}'.format(x=x, **course), 'coursebook.pdf'), source='#config.py', action=_download_pdf)
            xlsx = env.Command(target=os.path.join('{term}-{prefix}-{number}-{x}'.format(x=x, **course), 'coursebook.xlsx'), source='#config.py', action=_download_xlsx)
            csv0 = env.Command(target=os.path.join('{term}-{prefix}-{number}-{x}'.format(x=x, **course), 'coursebook.csv'), source=xlsx, action=_xlsx_to_csv)
            ppm = env.PDFtoPPM(source=[pdf, csv0])
            yield pdf
            yield xlsx
            yield csv0
            yield ppm
            for y in ppm:
                yield env.PPMtoPNG(source=[y, csv0])
        if len(course['section']) > 1:
            S = ['#config.py'] + [os.path.join('{term}-{prefix}-{number}-{x}'.format(x=x, **course), 'coursebook.csv') for x in course['section']]
            csv1 = env.Command(target='{term}-{prefix}-{number}.csv'.format(**course), source=S, action=_section_csv)

X = list(g())

import shutil

def _cleanup(target, source, env):
    if DRIVER is not None:
        _wait(8, 5)
        DRIVER.quit()
    if os.path.exists('download'):
        shutil.rmtree('download')

env.Command(target='cleanup', source=X, action=Action(_cleanup, 'Cleaning up.'))

