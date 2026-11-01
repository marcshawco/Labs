# Lab 1: Azure fundamentals

Build the smallest useful thing in Azure and follow one file all the way through it. Create a resource group, create a storage account inside it, push a text file up, edit that file in the browser, pull it back down, and confirm the change survived the round trip.

The point is not the file. The point is understanding how Azure nests things, because every later lab in this repo sits inside the same structure.

| | |
|---|---|
| Portal | portal.azure.com |
| Resource group | `Lab-Test-Marcus` |
| Region | West US 2 |
| Storage account | `marcusshawstorage`, Standard tier, locally redundant |
| Container | `upload-blob`, private access |
| Cost | A few cents. Storage is cheap; VMs are what cost money. |

## How Azure nests

Tenant, then subscription, then resource group, then resources. A resource group is a folder. Delete the folder and everything in it goes too, which is the cleanup mechanism I used at the end of every standalone lab in this repo.

At work the subscription boundary usually tracks a department or a cost centre, which is how finance attributes spend. I only have one.

## Build

Resource groups blade, empty, which is where every subscription starts.

![Empty resource groups blade](images/01-resource-groups-empty.png)

Creating the group. Region matters here: it decides which Microsoft datacenter physically holds the metadata, and later it decides which VMs can attach to which virtual networks.

![Create resource group, basics tab](images/02-create-rg-basics.png)

I added tags, which the lab did not ask for. Tags are key/value metadata and in a real environment they are how you answer "who owns this and who pays for it" six months later.

![Resource group tags](images/03-create-rg-tags.png)

![Review and create](images/04-create-rg-review.png)

![Resource group created](images/05-rg-created.png)

Group exists, nothing in it yet.

![Empty resource group overview](images/06-rg-overview-empty.png)

Next the storage account. The portal search bar is faster than hunting through the menu.

![Searching for storage in the portal](images/07-portal-search-storage.png)

![No storage accounts yet](images/08-storage-accounts-empty.png)

Two settings worth understanding on this screen. The storage account name has to be globally unique across all of Azure, not just my subscription, because it becomes a DNS name. And redundancy decides how many copies of my data exist and where:

- Locally redundant (LRS): several copies inside one datacenter. Cheapest. What I chose.
- Zone redundant (ZRS): copies across availability zones in one region.
- Geo redundant (GRS): copies in a second region hundreds of miles away.

Lab data does not need to survive a regional outage, so LRS.

![Create storage account](images/09-create-storage-account.png)

![Deployment in progress](images/10-deployment-in-progress.png)

![Deployment complete](images/11-deployment-complete.png)

Back in the resource group, the storage account now shows up inside it. This is the nesting working as expected.

![Resource group containing the storage account](images/12-rg-with-storage-account.png)

## The round trip

A plain text file on my Mac, written in TextEdit set to plain text.

![Local test file](images/13-local-test-file.png)

A container inside the storage account. Think of it as a folder inside the drive. I left anonymous access disabled, so the container is private.

![Creating a blob container](images/14-create-blob-container.png)

"Blob" is binary large object, which in practice just means file.

![Upload blob pane](images/15-upload-blob-pane.png)

![File selected for upload](images/16-upload-blob-file-selected.png)

Uploaded, 39 bytes, block blob, hot tier.

![Blob uploaded and listed](images/17-blob-uploaded.png)

Editing it in the browser. I added two lines that did not exist in the local copy.

![Editing the blob in the portal](images/18-edit-blob-in-portal.png)

Then downloaded it and opened it locally. The portal edit is there.

![Downloaded file showing the portal edits](images/19-downloaded-file-with-edits.png)

Upload, server-side edit, download, verify. That is the whole loop, and it confirms I was editing the real object in Azure rather than a cached copy in the browser.

## Cleanup

Deleted the resource group, which took the storage account, the container, and the blob with it. Then checked Cost Management > Cost analysis to confirm nothing was still accruing.

Checking cost analysis after every lab became a habit. A forgotten VM is the expensive mistake in Azure, not a forgotten text file.

## What I can speak to

- The tenant / subscription / resource group / resource hierarchy, and why resource groups are the unit of cleanup
- Why storage account names are globally unique (they resolve as DNS names)
- The redundancy options and the actual failure each one protects against
- Tagging as a cost attribution tool
- Verifying a change round-tripped rather than assuming the UI told the truth

---

[Back to all labs](../../README.md) · Next: [Network traffic analysis](../02-network-traffic-analysis/)
