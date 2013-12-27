## Warts

I'm still not completely happy with how state is handled. In contrast
dealing with paths has become considerably cleaner with the inclusion
of the `ICursor` types - users can simply access their application
data using the ClojureScript standard library and we can track path
under the hood.

The state tension lies in the fact that we components to be shareable
and in order to be shareable they need to be oblivious as to where in
the UI state their actual information presides.
