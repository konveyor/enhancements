# MTC Progress Reporting Improvements

The goal of this document is to highlight user experience issues in current implementation of progress reporting in MTC, and propose improvements in some of the areas for a better user experience.

## How progress reporting looks like in MTC?

The Status field in the _MigMigration_ CR contains the information about the progress of ongoing migration. Upon creation of _MigMigration_ CR, the migration transitions from one _Phase_ to another until it reaches the _Completed_ phase. For a user, a _Phase_ is simply a _Step_ among different steps that are involved in the migration. 


