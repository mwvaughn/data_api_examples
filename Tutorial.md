# 

## Set up a project directory

The following will set up a git repository capable of holding code for multiple related APIs: 
```
mkdir data_api_examples
cd data_api_examples
git init
echo "# Example Araport Data APIs" > README.md
for C in query generic passthrough
do
mkdir $C
touch $C/metadata.yml
touch $C/main.py
echo "# Example $C API" > $C/README.md
done

curl -skL "http://bit.ly/1yS7eX5" -s -o ".gitignore"

git add .
git commit -m "First commit"
```

## Connect your local repository to Github

* Go to the [Github repository creation page](https://github.com/new)
* Create a new repo called "data_api_examples". Make sure it is Public. Don't bother to initialize it.
* Make a note of the URL for the new repository
  * In my case it is https://github.com/mwvaughn/data_api_examples.git
* Enter the following in your command line from within your local git repo

```
# Sub in the URL for your Github repo in place of mine!
git remote add origin https://github.com/mwvaughn/data_api_examples.git
git push -u origin master
```

You should be able to go to your Github and see the newly created files!

## A service to access the MapMan bins API

Behind the scenes of our API, we will be issuing a request modelled after the following:

```
GET www.gabipd.org/services/rest/mapman/bin?request=[{"agi":"At4g25530"}]
```

And we expect a response similar to this:

```
[
    {"request":
        {"agi":"At4g25530"},
     "result":[
        {"code":"27.3.22",
         "name":"RNA.regulation of transcription.HB,Homeobox transcription factor family",
         "description":"no description",
         "parent":
            {"code":"27.3",
             "name":"RNA.regulation of transcription",
             "description":"no description",
             "parent":
                {"code":"27",
                 "name":"RNA",
                 "description":"no description",
                 "parent":null}}}]}]
```

This response, while not conforming to the AIP loose standard, is valid JSON so we'll pass it back to consumers. We will, however, normalize the inputs to use the AIP "locus" parameter name instead of accepting a JSON object. We will implement a "generic" type API to accomplish these twin objectives. 

### Create a generic API

Open up data_api_examples/generic/main.py in your editor and paste in the following. At minimum, you need to define required Python modules plus two functions: search() and list(). These three modules let you make HTTP requests, work with JSON, and make use of regular expressions - quite handy! 

```
import requests
import json
import re

def search(arg):
    return

def list(arg):
    return
```

Let's implement handling of the 'locus' parameter. The ADAMA service that powers our community APIs will eventually take on validation of parameters but for now, if you want robust services you need to add in some error handling. Edit your search() function so it looks like the following:

```
def search(arg):

# arg contains a dict with a single key:value
# locus is AGI identifier and is mandatory
    
    # Return nothing if client didn't pass in a locus parameter
    if not ('locus' in arg):
        return
    
    # Validate against a regular expression
    locus = arg['locus']
    locus = locus.upper()
    p = re.compile('AT[1-5MC]G[0-9]{5,5}', re.IGNORECASE)
    if not p.search(locus):
        return
        
    print locus
```

Now, let's go test it out in a Python environment:

```
Python 2.7.6 (default, Sep  9 2014, 15:04:36) 
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.39)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import main
>>> main.search({'locus':'AT4G25530'})
AT4G25530
>>> main.search()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: search() takes exactly 1 argument (0 given)
>>> main.search({'foo':'AT4G25530'})
>>> main.search({'locus':'bar'})
>>>
```

A few notes:
1. We invoke the main's search method with a python dict. This emulates the way the ADAMA server sends in parameters.
2. The code validation checks for the existence of the locus key and returns if not present
3. The code validation checks for a valid locus name and returns if not found
4. If one fails to pass in a dict to search(), there is also a failure response

We're ready now to wire up an HTTP request to the MapMan remote service. We'll use a simple form of the Python requests module here, but if you're not familiar with it you may want to [check out its documentation in detail](http://docs.python-requests.org/en/latest/). Update your search() function to match this example.

```
def search(arg):

# arg contains a dict with a single key:value
# locus is AGI identifier and is mandatory
    
    # Return nothing if client didn't pass in a locus parameter
    if not ('locus' in arg):
        return
    
    # Validate against a regular expression
    locus = arg['locus']
    locus = locus.upper()
    p = re.compile('AT[1-5MC]G[0-9]{5,5}', re.IGNORECASE)
    if not p.search(locus):
        return

    # Construct an access URL for the MapMan API
    param = '[{%22agi%22:%22' + locus + '%22}]'
    r = requests.get('http://www.gabipd.org/services/rest/mapman/bin?request=' + param)
    
    # Debug: Print the headers and text response
    print r.headers['Content-Type']
    print r.text
```

Go test it out in a Python environment:

```
>>> import main
>>> main.search({'locus':'AT4G25530'})
application/json
[{"request":{"agi":"AT4G25530"},"result":[{"code":"27.3.22","name":"RNA.regulation of transcription.HB,Homeobox transcription factor family","description":"no description","parent":{"code":"27.3","name":"RNA.regulation of transcription","description":"no description","parent":{"code":"27","name":"RNA","description":"no description","parent":null}}}]}]
```

We're now ready to return the response to the client. In a generic API, we do this by returning a valid content-type and the content of the HTTP response. Modify your code to look like this:

```
def search(arg):

# arg contains a dict with a single key:value
# locus is AGI identifier and is mandatory
    
    # Return nothing if client didn't pass in a locus parameter
    if not ('locus' in arg):
        return
    
    # Validate against a regular expression
    locus = arg['locus']
    locus = locus.upper()
    p = re.compile('AT[1-5MC]G[0-9]{5,5}', re.IGNORECASE)
    if not p.search(locus):
        return

    param = '[{%22agi%22:%22' + locus + '%22}]'
    r = requests.get('http://www.gabipd.org/services/rest/mapman/bin?request=' + param)
    
    #print r.headers['Content-Type']
    #print r.text
    
    if r.ok:
        return r.headers['Content-Type'], r.content
    else:
        return 'text/plaintext; charset=ISO-8859-1', 'An error occurred on the remote server'
```

Technically, you can just return the header provided by the remote service and its content, but we are clever and check for 200 OK, returning a plaintext error if we don't see it in the remote server response. 

The service is working locally. We're ready to register it as an Araport community API. First, we need to supply some metadata. Edit your generic/metadata.yml file to to match this example:

```
--- 
description: "Returns MapMan bin information for a given AGI locus identifier"
main_module: generic/main.py
name: mapman_bin_by_locus
type: generic
url: "http://www.gabipd.org/"
version: 0.1
whitelist: 
  - www.gabipd.org
```

Drop into your command line and commit all your changes, then push them up to the remote repository.

```
git add .
git commit -m "Working version of a generic API for MapMan"
git push
```

### Interact with the community API to create a new service with your code.

We use these ENV variables to make life easier. Go ahead and make sure they are set in your current terminal.
```
export GITHUB_UNAME=*YOUR GITHUB USER NAME*
export NS=$GITHUB_UNAME
export API=https://api.araport.org/community/v0.3
```

Assuming you have installed and configured it following instructions [here], use the Agave CLI to grab an OAuth2 token

```
auth-tokens-create -S
> Token successfully refreshed and cached for 14400 seconds
> c1abb7c8326c33483e4a8a39a918ed7
export TOKEN=c1abb7c8326c33483e4a8a39a918ed7
```

Post your new service into your namespace. Because we are creating multiple APIs from a single repo, we must specify the path (relative to the root of the repo) where the metadata.yml file can be found. If this is ommitted, ADAMA looks at the root level. 

```
curl -skL -XPOST -H "Authorization: Bearer $TOKEN" -F "git_repository=https://github.com/mwvaughn/data_api_examples.git" -F "metadata=generic" $API/$NS/services 

{
    "message": "registration started", 
    "result": {
        "list_url": "https://api.araport.org/community/v0.3/mwvaughn/mapman_bin_by_locus_v0.1/list", 
        "notification": "", 
        "search_url": "https://api.araport.org/community/v0.3/mwvaughn/mapman_bin_by_locus_v0.1/search", 
        "state_url": "https://api.araport.org/community/v0.3/mwvaughn/mapman_bin_by_locus_v0.1"
    }, 
    "status": "success"
}
```


Verify that it loaded correctly by accessing *state_url*

```
curl -skL -XGET -H "Authorization: Bearer $TOKEN" https://api.araport.org/community/v0.3/mwvaughn/mapman_bin_by_locus_v0.1

{
    "result": {
        "service": {
            "code_dir": "/tmp/tmpBMmBPk/user_code", 
            "description": "Returns MapMan bin information for a given AGI locus identifier", 
            "json_path": "", 
            "language": "python", 
            "main_module": "generic/main.py", 
            "metadata": "generic", 
            "name": "mapman_bin_by_locus", 
            "namespace": "mwvaughn", 
            "notify": "", 
            "requirements": [], 
            "self": "https://api.araport.org/community/v0.3/mwvaughn/mapman_bin_by_locus_v0.1", 
            "type": "generic", 
            "url": "http://www.gabipd.org/", 
            "version": 0.1, 
            "whitelist": [
                "www.gabipd.org", 
                "129.114.97.2", 
                "129.114.97.1", 
                "129.116.84.203"
            ], 
            "workers": [
                "7f3f1a32bbdeea510b5491ce03755dedda4efe7ed921e7632afb44b6f9e09adb"
            ]
        }
    }, 
    "status": "success"
}
```

Test the service by issuing a query to it. Note the presence of the "search" endpoint!

```
curl -skL -XGET -H "Authorization: Bearer $TOKEN" https://api.araport.org/community/v0.3/vaughn-dev/mapman_bin_by_locus_v0.1/search?locus=At4g25530

[{"request":{"agi":"AT4G25530"},"result":[{"code":"27.3.22","name":"RNA.regulation of transcription.HB,Homeobox transcription factor family","description":"no description","parent":{"code":"27.3","name":"RNA.regulation of transcription","description":"no description","parent":{"code":"27","name":"RNA","description":"no description","parent":null}}}]}]
```

