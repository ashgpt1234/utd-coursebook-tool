
import os

env = Environment()

k = 'DISPLAY'
v = os.environ.get(k)
if v:
    env['ENV'][k] = v

SConscript('SConscript', exports=['env'])
 
