## 기술 문서 정리

CS, Language, Infra .. 등등 필요한 문서들 정리

https://ljy2855.github.io/obsidian.md/


```dataview
TABLE file.mtime as Modified, file.folder AS "분류", (Done ? "Done" : "Writing") as Done SORT file.path desc
```


```dataviewjs
dv.table(["Note","Modified", "분류", "Done"], dv.pages("") .map(page => [ 
page.file.link, page.file.mtime, page.file.folder, page.Done ? "✔️" : "❌" ]) );
```