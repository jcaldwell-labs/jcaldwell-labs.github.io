---
title: "Rethinking the Terminal: Spatial Navigation in a Linear World"
description: "How my-grid brings infinite canvas concepts to the command line, inspired by Jef Raskin's vision of navigating to information instead of opening windows"
date: 2026-02-04
categories: ["tools"]
tags: ["terminal", "productivity", "vim", "devops", "ascii-art"]
---

The terminal is linear. Commands scroll up. Output disappears. Information exists only in the moment.

But what if it didn't have to be this way?

---

## The Problem with Traditional Terminals

I spend most of my day in a terminal. SSH sessions, log tailing, git operations, monitoring scripts. The workflow is always the same: run a command, read the output, run another command. Previous context scrolls away into oblivion.

To cope, we invented tools:
- **tmux** splits the screen into panes
- **watch** refreshes commands periodically
- **screen** keeps sessions alive
- Multiple terminal windows scattered across monitors

But these are workarounds, not solutions. They treat the symptom (losing context) without addressing the cause (linear information flow).

Then I read Jef Raskin's *The Humane Interface*.

---

## "Navigate to Information, Don't Open Windows"

Raskin, who led the original Macintosh project, proposed a radical idea: instead of opening windows to find information, you should *navigate* to where information lives. Imagine a vast plane of content where everything has a location. You zoom and pan to find what you need. The information doesn't come to you—you go to it.

This is how people naturally organize physical space. Papers on a desk have positions. Books on a shelf have locations. Your brain remembers *where* things are, not what window they're in.

What if terminals worked the same way?

---

## Enter my-grid

**my-grid** is a spatial canvas editor for the terminal. Instead of linear output, you get an infinite 2D plane where you can:

- Draw and diagram directly in ASCII
- Place live terminals anywhere on the canvas
- Monitor commands that auto-refresh
- Organize everything spatially with bookmarks

Think of it as **tmux + vim + infinite whiteboard + system monitoring** in a single tool.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         my-grid canvas                               │
│                                                                      │
│   ┌─────────────────┐        ┌─────────────────────────────────┐   │
│   │  DISK USAGE     │        │         LIVE TERMINAL           │   │
│   │  watch df -h    │        │         (PTY zone)              │   │
│   │  ─────────────  │        │  $ git status                   │   │
│   │  /dev/sda1 45%  │        │  On branch main                 │   │
│   │  /dev/sdb1 12%  │        │  nothing to commit              │   │
│   └─────────────────┘        │  $ _                            │   │
│                              └─────────────────────────────────┘   │
│   ┌─────────────────┐                                               │
│   │  GIT STATUS     │        ┌─────────────────────────────────┐   │
│   │  watch 5s       │        │           NOTES                 │   │
│   │  ─────────────  │        │  - Review PR #123               │   │
│   │  M  src/app.py  │        │  - Deploy staging               │   │
│   │  ?? newfile.txt │        │  - Check monitoring             │   │
│   └─────────────────┘        └─────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

This isn't a mockup. It's an actual workspace you can build in seconds.

---

## The Zone System

The killer feature is **zones**—rectangular regions with dynamic behavior:

| Zone Type | What It Does | Use Case |
|-----------|--------------|----------|
| **PIPE** | Run a command once, display output | `tree`, `ls`, snapshots |
| **WATCH** | Re-run command on interval | `df -h`, `git status` |
| **PTY** | Full interactive terminal | Shell, vim, Python REPL |
| **SOCKET** | Listen on TCP port | Remote control, API |
| **FIFO** | Listen on named pipe | Inter-process communication |

Creating a DevOps dashboard takes four commands:

```bash
:zone watch DISK 40 10 30s df -h
:zone watch GIT 50 15 5s git status --short
:zone watch LOGS 100 25 2s tail -20 /var/log/syslog
:zone pty TERM 80 24
```

Now you have disk monitoring, git status, log tailing, and an interactive terminal—all visible simultaneously, all spatially organized, all with vim-style navigation.

---

## Why Spatial Organization Matters

The magic isn't the individual features. It's the combination.

**Memory is spatial.** After using my-grid for a few sessions, you develop muscle memory: "disk usage is up and to the left, terminal is right, notes are bottom-right." You stop *looking* for information and start *going* to it.

**Context persists.** Unlike scrolling terminal output, your canvas stays put. Draw a diagram while monitoring logs. Add notes next to the code you're debugging. Everything coexists.

**Bookmarks are instant.** Press `m` + letter to save a position. Press `'` + letter to jump back. 36 bookmark slots (a-z, 0-9) mean you can teleport across your workspace.

**Layouts are reusable.** Save your zone configuration as a layout. Load it tomorrow. Share it with your team.

---

## Real Use Cases

### DevOps Monitoring

My daily monitoring setup:

```bash
:layout load devops
```

This loads a pre-configured workspace with:
- System resource graphs (watch zones)
- Kubernetes pod status
- Application logs
- An interactive terminal for commands
- Notes zone for incidents

Everything updates in real-time. Everything has a place.

### Development Workspace

When coding:

```bash
:zone pty EDITOR 100 40 vim
:zone watch TESTS 50 12 5s pytest --tb=short
:zone pipe TREE 40 20 tree -L 2 src/
:zone create NOTES 0 0 40 15
```

The editor gets most of the space. Tests auto-run and show results. Project structure is always visible. Notes persist across sessions.

### ASCII Diagramming

Sometimes you just need to draw a diagram:

```bash
:rect 60 30           # Draw a rectangle
D                     # Enter draw mode
:box stone "API"      # Create labeled box
:figlet SYSTEM        # ASCII art header
```

my-grid becomes a diagramming tool with box-drawing characters, figlet integration, and the ability to mix diagrams with live data.

---

## Technical Implementation

Under the hood, my-grid uses:

**Sparse canvas storage** — Only non-empty cells consume memory. The canvas is theoretically infinite in all directions. Store a million characters at position (1000000, 1000000)? No problem—it's just a dictionary lookup.

```python
@dataclass
class Canvas:
    _cells: dict[tuple[int, int], Cell] = field(default_factory=dict)

    def get(self, x: int, y: int) -> Cell:
        return self._cells.get((x, y), Cell())
```

**PTY zones with full terminal emulation** — Uses the `pyte` library for proper VT100/ANSI handling. Colors, cursor control, scrollback—it all works.

**Socket API for automation** — Control my-grid from external scripts:

```bash
python mygrid.py --server --port 8765
echo ':rect 10 5' | nc localhost 8765
```

Pipe CI/CD output to the canvas. Build custom dashboards. Integrate with anything that can send TCP.

---

## The Philosophy

my-grid isn't trying to replace vim or tmux. It's exploring a different question: **What if terminal interfaces were spatial instead of temporal?**

Traditional terminals optimize for sequential interaction. Command, response, command, response. my-grid optimizes for *situational awareness*—multiple information streams visible simultaneously, organized by meaning rather than recency.

Raskin wrote about "zooming user interfaces" where you navigate a vast information plane. my-grid is a modest implementation of that idea, constrained to ASCII but surprisingly powerful because of it.

---

## Getting Started

```bash
git clone https://github.com/jcaldwell-labs/my-grid.git
cd my-grid
pip install -r requirements.txt
python mygrid.py
```

Start with these commands:
- `wasd` or arrows to move
- `i` to enter edit mode (type text)
- `:rect 20 10` to draw a rectangle
- `:zone watch STATUS 40 10 5s uptime` to create a watch zone
- `:w myproject.json` to save

The learning curve is similar to vim—steep initially, then incredibly efficient.

---

## What's Next

Current roadmap includes:
- Mouse support for zone interaction
- WebSocket zones for browser integration
- Collaborative editing over network
- Plugin system for custom zone types

But the core is solid. I use it daily for DevOps monitoring, and it's changed how I think about terminal workflows.

The terminal doesn't have to be linear. Information doesn't have to scroll away. Sometimes the old ideas—navigate to information, organize spatially, remember locations—are exactly what modern tools need.

---

*my-grid is open source under MIT license. Find it at [github.com/jcaldwell-labs/my-grid](https://github.com/jcaldwell-labs/my-grid).*
