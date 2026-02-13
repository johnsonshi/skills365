# Tools and Clients Best Practices

## Tool Selection Guide

| Use Case | Tool | Why |
|----------|------|-----|
| Ad-hoc exploration | Web UI | Quick, no install |
| Complex development | Kusto.Explorer | Full IDE features |
| Automation/CI-CD | Kusto.Cli | Scriptable |
| Local testing | Emulator | Free, offline |
| Custom apps | Monaco-Kusto | Embeddable |

---

## Kusto.Explorer Keyboard Shortcuts

### Execution
| Action | Shortcut |
|--------|----------|
| Run query | `F5` or `Shift+Enter` |
| Cancel | `Esc` or `Shift+F5` |
| Preview results | `Ctrl+F5` |

### Editing
| Action | Shortcut |
|--------|----------|
| Format query | `Alt+Shift+F` |
| Comment | `Ctrl+K, Ctrl+C` |
| Uncomment | `Ctrl+K, Ctrl+U` |
| Rename symbol | `Ctrl+R, Ctrl+R` |
| Extract to let | `Alt+Ctrl+M` |
| Go to definition | `F12` |

### Visualization
| Action | Shortcut |
|--------|----------|
| Column chart | `Alt+C` |
| Time chart | `Alt+T` |
| Pie chart | `Alt+P` |

---

## Web UI Keyboard Shortcuts

| Action | Windows | Mac |
|--------|---------|-----|
| Run query | `Shift+Enter` | `Shift+Enter` |
| Cancel | `Ctrl+Shift+Enter` | `Cmd+Shift+Enter` |
| New tab | `Ctrl+J` | `Cmd+J` |
| Format | `Shift+Alt+F` | `Shift+Option+F` |
| Copy link | `Shift+Alt+L` | `Shift+Option+L` |

---

## Productivity Tips

### Kusto.Explorer
1. **Use Search++ Mode** for initial data exploration
2. **Check Issues tab** after queries for optimization suggestions
3. **Rename tabs** (`Ctrl+F2`) for organization
4. **Export connections** before reinstalling

### Web UI
1. **Pin frequently used clusters** for quick access
2. **Use Recall** to find previous queries
3. **Create dashboards** for repeated analysis
4. **Share via deep links** for collaboration

### Kusto.Cli
1. **Use `-lineMode:false`** for multi-line queries in scripts
2. **Set `-scriptQuitOnError:true`** for CI/CD pipelines
3. **Use `#save`** directive before queries to export results
4. **Capture output** with `-transcript:<file>`

### Emulator
1. **Allocate sufficient memory** (`-m 4G` minimum)
2. **Use volumes** for persistent data
3. **Test queries locally** before production
4. **Validate schema changes** in isolation

---

## Environment Recommendations

### Development
- Primary: Kusto.Explorer
- Testing: Emulator
- Custom tools: Monaco-Kusto

### Production Monitoring
- Primary: Web UI dashboards
- Investigation: Kusto.Explorer
- Automation: Kusto.Cli

### CI/CD
- Testing: Kusto.Cli + Emulator
- Validation: Script-based verification
- Deployment: Kusto.Cli for schema changes
