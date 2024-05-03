## 기술 문서 정리

CS, Language, Infra .. 등등 필요한 문서들 정리

https://ljy2855.github.io/obsidian.md/


```dataviewjs
dv.table(["Note","Modified", "Category", "Done"], dv.pages("") .map(page => [ 
page.file.link, page.file.mtime, page.file.folder, page.Done ? "✔️" : "❌" ]) );
```