# Worker Workflow

Use one feature branch for each discrete piece of work. Keep the branch focused
on its feature, and commit changes with a clear message.

Before integrating a completed feature, rebase its branch onto the latest
`main` if there is a merge conflict. Resolve any conflicts in the feature branch,
then run the relevant checks again.

After the feature is reviewed and its checks pass, merge it into `main` with a
squash merge. The resulting `main` commit should describe the completed feature
instead of the branch's intermediate commits.

Do not combine unrelated work in the same feature branch or squash merge.
