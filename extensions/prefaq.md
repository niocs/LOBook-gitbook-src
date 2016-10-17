# This part has some Q&A so that readers understand some basics.

1. Can there be multple soffice processes per user per host ?

    Answer : There can only be one soffice process per user per host, if one soffice process is running and you try to open a document with another soffice command via terminal, this request will be transfered to the already running soffice process and the document will be opened in a new window of the same process. However while an instance of soffice gui instance any number of `headless` soffice commands can be run. This is probably because the headless processes need only readonly access to source files and cfg.

2. Can there be extensions installed just for a particular document or can there be any correspondence between extensions and the documents open by soffice ?

   Answer : An installed extension is common to all documents opened by soffice, there is no concept of per document extension. The extension should be prepared to handle the presence of multiple documents/ or calls from them.

3. Is there any easy method to see the states of calc process while running without using gdb ?

   Answer : As far as we know there is no such way other than using gdb.

