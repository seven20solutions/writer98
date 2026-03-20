New Share menu contains:
- Server
- Collab code
- Publish

Server displays a modal which allows you to enter the endpoint for a Cloudflare Worker server. This Worker needs to help deliver the functionality for the next two tools. This setting is persistent and can also be defined within a query string ?s=myendpoint.com

Collab Code is a shareable string the user defines such as “Apple-Pear-Banana”. It is a persistent value and should be associated with the other data saved with the current document. It should also be included in the export. 

By using a Collab Code, it connects the changes you are making to the Worker whereby it updates the Worker with the document changes at defined intervals. This allows you to give another person the server value and Collab Code, effectively sharing the document with them. When they connect to the Worker, it will identify them as a different editor and pull down a copy of the document for them to edit. As they edit, the Worker will be updated, and their changes will appear on all other connected versions of the document. The Collab Code can be defined in a query string: ?cc=Apple-Pear-Banana by sending a user a link with the CC and Server values, you effectively send them the document to collaborate on with you. 

Publish is similar but it keeps the document read-only. Recipients are send a publish key that allows them to see but not edit the document. ?pk=ejkejenejdiirrr

The publish key is bound to the document and does not change. The user can’t define it; instead, it needs to be randomly generated when the document is published to the server. This happens when the user clicks Publish. They see a modal with the share link. The modal also allows them to unshare / unpublish the document which effectively prevents any existing publish links for that document from working anymore. 

Documents which have a Collab Code or a Publish Key should get little icons next to their title on the Open modal so the user can see how that document is configured. 

Also, when the user edits a file which is connected to the server, the status bar should reflect when the document has pushed changes to the server and pulled edits from another user. 
