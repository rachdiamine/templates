# Production Templates (Reference Only)

## Important Notice

⚠️ **This folder contains PRODUCTION templates that should match the Git repository at https://github.com/rachdiamine/templates**

### For Developers

**DO NOT use these templates for local development!**

For local development, use the `dev-templates/` folder inside `phala-cloud-node-starter/`:

```
phala-cloud-node-starter/dev-templates/
├── registry-simple.json
└── web-templates/
    ├── tanzania_tcra_witness_V4
    ├── icasa_witness_v2
    └── ...
```

### Purpose of This Folder

This `templates/` folder serves as:

1. **Production Reference**: These templates should match what's deployed in the Git repository
2. **Git Sync Source**: This is the folder that should be synced to https://github.com/rachdiamine/templates
3. **Production Truth**: Production TEE servers fetch templates from Git, not from this folder

### Workflow

```
┌─────────────────────────────────────────────────────────────┐
│  Development (Local)                                        │
│  - Edit: phala-cloud-node-starter/dev-templates/            │
│  - Server loads from: dev-templates/                        │
│  - TEMPLATE_SOURCE=local                                    │
└─────────────────────────────────────────────────────────────┘
                          │
                          ↓ When template is stable
┌─────────────────────────────────────────────────────────────┐
│  Production Sync                                            │
│  - Copy: dev-templates/ → templates/                        │
│  - Commit and push to Git: templates/ → GitHub              │
└─────────────────────────────────────────────────────────────┘
                          │
                          ↓ Automatic
┌─────────────────────────────────────────────────────────────┐
│  Production (Phala Cloud)                                   │
│  - Server fetches from: GitHub repository                   │
│  - Source: https://github.com/rachdiamine/templates         │
│  - TEMPLATE_SOURCE=git                                      │
└─────────────────────────────────────────────────────────────┘
```

### Syncing Templates to Production

When you're ready to deploy a template to production:

1. **Test locally** with `dev-templates/`
2. **Copy to production folder**:
   ```bash
   cp phala-cloud-node-starter/dev-templates/registry-simple.json templates/
   cp -r phala-cloud-node-starter/dev-templates/web-templates/* templates/web-templates/
   ```
3. **Commit and push** this `templates/` folder to the Git repository
4. **Production servers** will automatically fetch the updated templates on next request

### Directory Structure Comparison

```
Ghostwitness/
├── templates/                                    ← YOU ARE HERE (prod reference)
│   ├── registry-simple.json                     ← Synced to Git
│   └── web-templates/                           ← Synced to Git
│
├── phala-cloud-node-starter/
│   ├── dev-templates/                           ← USE THIS for development
│   │   ├── registry-simple.json                 ← Edit freely
│   │   └── web-templates/                       ← Edit freely
│   ├── src/index.ts                             ← TEE server code
│   └── .dockerignore                            ← Excludes dev-templates from builds
```

### Key Points

- ✅ `dev-templates/` tracked in Git (developers can share)
- ✅ `dev-templates/` excluded from Docker builds (not bundled)
- ✅ `templates/` tracked in Git (production reference)
- ✅ `templates/` synced to separate GitHub repo
- ✅ Production fetches from GitHub, not local files

---

**Remember:** Always test in `dev-templates/` first, then promote to `templates/` when ready for production!
