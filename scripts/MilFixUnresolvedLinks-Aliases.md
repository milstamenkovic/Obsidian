<%*

// Mil Fix Unresolved Links-Aliases
// 2 of 2 for Uresolved Links-Aliases Script for Obsidian
// Created by Mil Stamenković
// © 2026 Mil Stamenković
// Created with help of Free Claude AI
// Created: 25.02.2026. - Ćuprija, Serbia
// Revised: 25.02.2026. - Ćuprija, Serbia

// This SCRIPT is used to FIX all unresolved connections between LINKS in old notes and ALIASES in new notes.
// This is the 2nd (SECOND) step - It comes after "Preview Unresolved Links-Aliases" script!

// E.g. Old note about universe had PREEMPTIVE link [[black hole]] (singular) inside note text, when note about black holes didn't existed yet.
// Then new note about black holes was made called BLACK HOLES (plural) with alias BLACK HOLE (singular). This SCRIPT connects PREEMPTIVE old [[black hole]] with new alias "black hole".

// 2026 - This script is used with Obsidian plugin TEMPLATER.



const files = app.vault.getMarkdownFiles();
let totalFixed = 0;
let totalSkipped = 0;

for (const file of files) {
    let content = await app.vault.read(file);
    let modified = false;
    const unresolvedLinks = content.match(/\[\[([^\]|#]+)\]\]/g);
    
    if (!unresolvedLinks) continue;
    
    for (const link of unresolvedLinks) {
        const linkText = link.slice(2, -2).trim();
        
        const existingFile = app.metadataCache.getFirstLinkpathDest(linkText, file.path);
        if (existingFile) continue;
        
        for (const targetFile of files) {
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
                const fixedLink = `[[${targetFile.basename}|${linkText}]]`;
                
                const choice = await tp.system.suggester(
                    [`✅ Fix: ${fixedLink}`, `⏭️ Skip`],
                    ["fix", "skip"],
                    false,
                );
                
                if (choice === "fix") {
                    content = content.replace(link, fixedLink);
                    modified = true;
                    totalFixed++;
                } else {
                    totalSkipped++;
                }
                break;
            }
        }
    }
    
    if (modified) {
        await app.vault.modify(file, content);
    }
}

new Notice(`Done! Fixed ${totalFixed} links, skipped ${totalSkipped} links.`);
%>