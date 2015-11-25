Title: Finding duplicated files
Date: 2014-08-08 13:20
Author: jibreel
Category: python
Slug: finding-duplicated-files

[os.walk](https://docs.python.org/2/library/os.html#os.walk) makes it
relativly easy to go through all of the files in a directory, so there
is some neat code we can write using `defaultdict` to parse all files in any dir and
find all the files that are duplicates.

Here is one possible way to do it

	import os
	from hashlib import md5
	from collections import defaultdict


	files = defaultdict(list)
	for root, dirs, filenames in os.walk('.'):
	    for filename in filenames:
	        file_path = os.path.join(root, filename)
	        with open(file_path, 'rb') as f:
	            key = md5(f.read()).digest()
	            files[key].append(file_path)

	files = [x for x in files.values() if len(x) > 1]
	print files


We set the key for the dictionary to be the md5 hash, and append to it
all the files with the same hash.


`defaultdict()` is a pretty handy class
when you want there to be a default value for any key that doesn't
exist, as in this case we want a list, but it could be dict, list, set,
int(yes, even int) ...


For example by doing `files[key].append(file_path)`, if the key
exist it appends the `file_path`, if not it will just create an empty
list and then append the `file_path` to it :)


Bellow is a slightly more robust module for it, which reads files in
chunks so that it doesn't cause memory problems if the file is very big.


Save it as `duplicates.py` and call it with
a directory path
`python duplicates.py ~/path/to/folder`

and it will print all the files that are duplicates


<span class="highlight-red">Edit</span>: It now only hashes files that
have one or more equal files with same size. this is much faster.


	import os
	import sys
	from hashlib import md5
	from pprint import pprint
	from collections import defaultdict


	if len(sys.argv) < 2:
	    print 'please enter a dir'
	    sys.exit()
	path = sys.argv[1]


	def md5_file(file_path, block_size=2**20 * 60):
	    """ return md5 hash of a file, 2**20 is 1mb """
	    m = md5()
	    with open(file_path, 'rb') as f:
	        while True:
	            chunk = f.read(block_size)
	            if not chunk:
	                break
	            m.update(chunk)
	    return m.digest()


	file_sizes = defaultdict(list)
	for root, dirs, filenames in os.walk(path):
	    for filename in filenames:
	        file_path = os.path.join(root, filename)
	        file_sizes[os.path.getsize(file_path)].append(file_path)
	# list of files with one or more equel file size
	file_sizes = [x for x in file_sizes.values() if len(x) > 1]

	files = defaultdict(list)
	for paths in file_sizes:
	    for path in paths:
	        key = md5_file(path)
	        files[key].append(path)
	files = [x for x in files.values() if len(x) > 1]

	for duplicates in files:
	    print '\n\n_________________________________________'
	    pprint(duplicates)
	    print '_________________________________________'

	print '\n%s files with atleast one duplicate\n' % len(files)




[link for code
repository](https://github.com/spaceexperiment/utils/blob/master/duplicates.py)
