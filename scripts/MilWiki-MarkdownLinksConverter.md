<%*

// Mil Wiki-Markdown Links Converter
// Converter for Wiki and Markdown linkss Script of Obsidian
// Created by Mil Stamenković
// © 2026 Mil Stamenković
// Created with help of Free Claude AI
// Created: 12.03.2026. - Ćuprija, Serbia
// Revised: 12.03.2026. - Ćuprija, Serbia

// This SCRIPT is used to find all Wiki or Markdown links and convert them to oposite (Wiki to Markdown and Markdown to Wiki).
// It works for a single ACTIVE NOTE, for ACTIVE FOLDER (where Acrive note is) and for the WHOLE VAULT.

// 2026 - This script is used with Obsidian plugin TEMPLATER.


// ---- CONVERSION FUNCTIONS ----

// Decode %20 and other encoded characters in URLs
function decodeUrl(url) {
    try { return decodeURIComponent(url); } catch { return url; }
}

// Check if a file exists in the vault (strips #heading from url)
function fileExists(url) {
    const cleanPath = url.split('#')[0].trim();
    if (!cleanPath) return true; // pure heading link, skip
    return app.vault.getFiles().some(f => 
        f.path === cleanPath || 
        f.name === cleanPath ||
        f.basename === cleanPath
    );
}

// Markdown [text](url) → Wiki [[url|text]]
function markdownToWiki(content, warnings, fileName) {
    const lines = content.split('\n');
    const result = lines.map((line, i) => {
        // Skip real checkbox lines - [ ] and - [x] only
        if (/^[\s]*-\s*\[[ x]\]/.test(line)) return line;

        const isTableRow = /^\s*\|/.test(line);

        return line.replace(/\[([^\]]*)\]\(((?:[^()]+|\([^()]*\))*)\)/g, (match, text, url) => {
            // Skip all protocol links
            if (/^[a-zA-Z][a-zA-Z0-9+.-]*:\/\//.test(url)) return match;

            const cleanUrl = decodeUrl(url).replace(/\.md$/, '');

            // Validate file exists
            if (!fileExists(cleanUrl)) {
                warnings.push(`  Line ${i + 1}: ${match}`);
            }

            // In table rows, don't use | alias to avoid breaking table structure
            if (isTableRow) {
                return `[[${cleanUrl}]]`;
            }

            if (text === cleanUrl || text === '') {
                return `[[${cleanUrl}]]`;
            }
            return `[[${cleanUrl}|${text}]]`;
        });
    }).join('\n');
    return result;
}

// Wiki [[url|text]] → Markdown [text](url)
function wikiToMarkdown(content, warnings, fileName) {
    const lines = content.split('\n');
    const result = lines.map((line, i) => {
        const isTableRow = /^\s*\|/.test(line);
        return line.replace(/\[\[([^\]|]+)(?:\|([^\]]+))?\]\]/g, (match, url, text) => {
            // Skip external URLs and file:// links
            if (/^[a-zA-Z][a-zA-Z0-9+.-]*:\/\//.test(url)) return match;

            // In table rows, ignore alias — use url as display text if no alias
            const displayText = (!isTableRow && text) ? text : (text ? text : url);

            // Validate file exists
            if (!fileExists(url)) {
                warnings.push(`  Line ${i + 1}: ${match}`);
            }

            return `[${displayText}](${url})`;
        });
    }).join('\n');
    return result;
}

// Strip frontmatter from content, return it separately
function splitFrontmatter(content) {
    if (!content.startsWith('---')) return { frontmatter: '', body: content };
    const end = content.indexOf('\n---', 3);
    if (end === -1) return { frontmatter: '', body: content };
    const frontmatter = content.substring(0, end + 4);
    const body = content.substring(end + 4);
    return { frontmatter, body };
}

// Process a single file
async function processFile(file, convertFn, allWarnings) {
    const content = await app.vault.read(file);
    const { frontmatter, body } = splitFrontmatter(content);
    const warnings = [];
    const convertedBody = convertFn(body, warnings, file.name);
    if (warnings.length > 0) {
        allWarnings.push({ file: file.path, warnings });
    }
    const converted = frontmatter + convertedBody;
    if (converted !== content) {
        await app.vault.modify(file, converted);
        return true;
    }
    return false;
}

// Process multiple files
async function processFiles(files, convertFn, allWarnings) {
    let count = 0;
    for (const file of files) {
        if (file.extension === 'md') {
            const changed = await processFile(file, convertFn, allWarnings);
            if (changed) count++;
        }
    }
    return count;
}

// Create a report note in the vault
async function createReport(allWarnings, directionLabel, count) {
    const date = new Date().toLocaleString();
    let report = `# 🔗 Link Converter Report\n`;
    report += `**Date:** ${date}\n`;
    report += `**Conversion:** ${directionLabel}\n`;
    report += `**Files changed:** ${count}\n\n`;

    if (allWarnings.length === 0) {
        report += `## ✅ No issues found!\nAll converted links look good.\n`;
    } else {
        report += `## ⚠️ Suspicious links (${allWarnings.length} files)\n`;
        report += `These links could not be matched to an existing file in your vault.\n`;
        report += `Please review them manually.\n\n`;
        for (const w of allWarnings) {
            report += `### 📄 [[${w.file}]]\n`;
            report += w.warnings.join('\n') + '\n\n';
        }
    }

    const reportPath = `Link Converter Report.md`;
    const existing = app.vault.getAbstractFileByPath(reportPath);
    if (existing) {
        await app.vault.modify(existing, report);
    } else {
        await app.vault.create(reportPath, report);
    }
    new Notice(`✅ Done! Report saved to "${reportPath}"`);
}


// ---- MENU ----

const scopeChoice = await tp.system.suggester(
    ['📄 Active note', '📁 Active folder', '🗄️ Whole vault'],
    ['note', 'folder', 'vault']
);
if (!scopeChoice) return;

const directionChoice = await tp.system.suggester(
    ['Markdown → Wiki', 'Wiki → Markdown'],
    ['toWiki', 'toMarkdown']
);
if (!directionChoice) return;

const convertFn = directionChoice === 'toWiki' ? markdownToWiki : wikiToMarkdown;
const directionLabel = directionChoice === 'toWiki' ? 'Markdown → Wiki' : 'Wiki → Markdown';
const allWarnings = [];
let count = 0;

// ---- EXECUTE ----

if (scopeChoice === 'note') {
    const file = tp.file.find_tfile(tp.file.title);
    const changed = await processFile(file, convertFn, allWarnings);
    count = changed ? 1 : 0;

} else if (scopeChoice === 'folder') {
    const folderPath = tp.file.folder(true);
    const files = app.vault.getFiles().filter(f => f.path.startsWith(folderPath));
    count = await processFiles(files, convertFn, allWarnings);

} else if (scopeChoice === 'vault') {
    const confirm = await tp.system.suggester(
        ['✅ Yes, convert whole vault', '❌ Cancel'],
        [true, false]
    );
    if (!confirm) return;
    const files = app.vault.getFiles();
    count = await processFiles(files, convertFn, allWarnings);
}

await createReport(allWarnings, directionLabel, count);
%>