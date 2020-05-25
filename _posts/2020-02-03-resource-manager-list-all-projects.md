---
layout: post
title: GCP Resource Manager - listing everything
categories: [GCP, Resource Manager API, Python]
---

This post is focused on [Resource Manager](https://cloud.google.com/resource-manager), which is a very powerful service in GCP, that allows creating, modifying, deleting and listing GCP resources (in essence, all what a CRUD model needs to do) over HTTP. The API allows to manage different resources that build the hierarchical structure of projects and resources in GCP, such as Organizations, Folders, and Projects, among others.

Another important use of the Resource Manager is that it allows to set and retrieve the IAM settings on those resources.

However, this post is focused on retrieving information about our projects, when we want to list all of our projects under a resource.

## The "issue"

Although the API has all of the needed features to manage our projects via a REST interface, it's missing some higer level features to, for example, listing all the projects under an organization, no matter under which hiererchical branch are they into. 

There is a command that seems to help here, which is the [`gcloud projects list`](https://cloud.google.com/sdk/gcloud/reference/projects/list) on the Google Cloud SDK, however, as stated in the documentation:

> "[*The `gcloud projects list` command*] Lists all active projects, where the active account has Owner, Editor or Viewer permissions"

So that means that if we have visibility on more than one organization, we will see all the projects on the same list no matter on which organization they are.

We might think to use some flag to filter the organization that we want. Let's use the same command again, but this time we will filter by a specific Organization ID:

```
gcloud projects list \
    --filter 'parent.type="organization" AND parent.id="[MY-ORGANIZATION-ID]"' 
```

Note that the organization ID is an uniquely identifying number, not the name of the organization that we want to filter! We can find it through the console, or by using the following command `gcloud organizations list --filter 'displayName="[MY-ORGANIZATION-NAME]"'` and copying the ID from the output.

We will receive less projects on the response, in case that we had visibility on more than one organization, or if we had visibility to projects without organization, so we might think that the command did what we wanted.

However, there is an issue here:

**If the organization has folders, the command won't search for projects inside of them, therefore, if there is one or more folders under an organization, we will miss the projects that these folders contain, even if they are on the hierarchy under the organization!**

## The solution

As I mentioned before, although the API doesn't directly provide us with the specific method that we need, we can use its resources to create a quick way to do what we need. Since I like Python, we can do calls to the Resource Manager API by using the [Google API Python client](https://github.com/googleapis/google-api-python-client).

We will also need a way to authenticate our calls to the API, we can use the [Google Auth Python Library](https://github.com/googleapis/google-auth-library-python) libraries for that regard.

I recommend creating a [Service Account](https://cloud.google.com/iam/docs/service-accounts) to perform the authentication. We will need a Service Account that has [Organization-Level permissions](https://cloud.google.com/resource-manager/docs/access-control-org). We have to go to the [IAM page for our organization](console.cloud.google.com/projectselector2/iam-admin/iam?organizationId=0) and add our service account there with the permissions of Folder Viewer, Organization Viewer, and Browser.

Once that is done, you can download the Service Account `JSON` key locally and point out this environment variable to its path:

```
export GOOGLE_APPLICATION_CREDENTIALS=/absolute-path-to-my-key/key.json
```

Then, we can start to write our code with:

```python
from googleapiclient.discovery import build
import google.auth

credentials, _ = google.auth.default()
```

Now, we will need to use the `googleapiclient` module to create our API clients. We will need two, one for each version of the Resource Manager API:

- v2 to call folder-related methods (such as listing folders)
- v1 for the rest of the resources (organizations, projects)

The code will follow up like this:

```python
rm_v1_client = build('cloudresourcemanager', 'v1', credentials=credentials, cache_discovery=False)

rm_v2_client = build('cloudresourcemanager', 'v2', credentials=credentials, cache_discovery=False)
```

Now, in the code we have to put the **parent** that we want to use to list all of our projects. For example, if it's an organization, do this:

```python
PARENT_ID="organizations/[MY-ORGANIZATION-ID]"
```

If it's a folder, instead add this:

```python
PARENT_ID="folders/[MY-FOLDER-ID]"
```

Now, for the following steps, we will have to list all the projects on the parent, and then list all the folders under that parent. This way, we will be able to search for projects under every folder, and for *other* folders under every other folder. We can just loop through it, for this example we will list all the projects under a single organization.


The whole code can look like this:

```python
from googleapiclient.discovery import build
import google.auth

credentials, _ = google.auth.default()

# V1 is needed to call all methods except for the ones related to folders
rm_v1_client = build('cloudresourcemanager', 'v1', credentials=credentials, cache_discovery=False) 

# V2 is needed to call folder-related methods
rm_v2_client = build('cloudresourcemanager', 'v2', credentials=credentials, cache_discovery=False) 

ORGANIZATION_ID = '[MY-ORG-ID]'

def listAllProjects():
    # Start by listing all the projects under the organization
    filter='parent.type="organization" AND parent.id="{}"'.format(ORGANIZATION_ID)
    projects_under_org = rm_v1_client.projects().list(filter=filter).execute()

    # Get all the project IDs
    all_projects = [p['projectId'] for p in projects_under_org['projects']]

    # Now retrieve all the folders under the organization
    parent="organizations/"+ORGANIZATION_ID
    folders_under_org = rm_v2_client.folders().list(parent=parent).execute()

    # Make sure that there are actually folders under the org
    if not folders_under_org:
        return all_projects

    # Now sabe the Folder IDs
    folder_ids = [f['name'].split('/')[1] for f in folders_under_org['folders']]

    # Start iterating over the folders
    while folder_ids:
        # Get the last folder of the list
        current_id = folder_ids.pop()
        
        # Get subfolders and add them to the list of folders
        subfolders = rm_v2_client.folders().list(parent="folders/"+current_id).execute()
        
        if subfolders:
            folder_ids.extend([f['name'].split('/')[1] for f in subfolders['folders']])
        
        # Now, get the projects under that folder
        filter='parent.type="folder" AND parent.id="{}"'.format(current_id)
        projects_under_folder = rm_v1_client.projects().list(filter=filter).execute()
        
        # Add projects if there are any
        if projects_under_folder:
            all_projects.extend([p['projectId'] for p in projects_under_folder['projects']])

    # Finally, return all the projects
    return all_projects

if __name__=='__main__':
    print(listAllProjects())
```

By running this script, we will be able to search for any project inside the organization, no matter if the project is inside a folder, or if a folder is inside of another folder.

One use of this, is if we for example want to retrieve all the users that have visibility on the projects of our organization, or if we want to audit them and later make some change(s).
