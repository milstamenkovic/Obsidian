<%*

// Mil Preview Unresolved Links-Aliases
// 1 of 2 for Uresolved Links-Aliases Script of Obsidian
// Created by Mil Stamenković
// © 2026 Mil Stamenković
// Created with help of Free Claude AI
// Created: 25.02.2026. - Ćuprija, Serbia
// Revised: 25.02.2026. - Ćuprija, Serbia

// This SCRIPT is used to PREVIEW all unresolved connections between LINKS in old notes and ALIASES in new notes.
// This is the 1st (FIRST) step - After this comes the 2nd script to fix links - "Fix Unresolved Links-Aliases!

// E.g. Old note about universe had PREEMPTIVE link [[black hole]] (singular) inside note text, when note about black holes didn't existed yet.
// Then new note about black holes was made called BLACK HOLES (plural) with alias BLACK HOLE (singular). This SCRIPT connects PREEMPTIVE old [[black hole]] with new alias "black hole".

// 2026 - This script is used with Obsidian plugin TEMPLATER.



const files = app.vault.getMarkdownFiles();
const results = [];

for (const file of files) {
    const content = await app.vault.read(file);
    const unresolvedLinks = content.match(/\[\[([^\]|#]+)\]\]/g);
    
    if (!unresolvedLinks) continue;
    
    for (const link of unresolvedLinks) {
        const linkText = link.slice(2, -2).trim();
        
        const existingFile = app.metadataCache.getFirstLinkpathDest(linkText, file.path);
        if (existingFile) continue;
        
        for (const targetFile of files) {
            // Skip if source and target are the same note
            if (targetFile.path === file.path) continue;
            
            const cache = app.metadataCache.getFileCache(targetFile);
            if (!cache?.frontmatter?.aliases) continue;
            
            const aliases = Array.isArray(cache.frontmatter.aliases) 
                ? cache.frontmatter.aliases 
                : [cache.frontmatter.aliases];
            
            const matchingAlias = aliases.find(a => 
                a?.toLowerCase() === linkText.toLowerCase()
            );
            
            if (matchingAlias) {
                results.push({
                    sourceNote: file.path,
                    unresolvedLink: link,
                    suggestedFix: `[[${targetFile.basename}|${linkText}]]`
                });
                break;
            }
        }
    }
}

if (results.length === 0) {
    new Notice("✅ No unresolved alias links found!");
} else {
    const previewContent = `# Unresolved Alias Links Preview\nFound **${results.length}** links to fix.\n\n` +
        results.map(r => 
            `### 📄 ${r.sourceNote}\n- ❌ ${r.unresolvedLink}\n- ✅ ${r.suggestedFix}`
        ).join("\n\n");
    
    const existing = app.vault.getAbstractFileByPath("_alias_preview.md");
    if (existing) await app.vault.delete(existing);
    
    await app.vault.create("_alias_preview.md", previewContent);
    await app.workspace.openLinkText("_alias_preview", "", true);
    new Notice(`Found ${results.length} unresolved alias links!`);
}
%>