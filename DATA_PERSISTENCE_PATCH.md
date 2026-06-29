# EEL TeamChat - Data Persistence Fix

## Problem
When users create a group chat and add workers, all data is wiped when the app is closed.

## Solution
Implement localStorage persistence with backup/restore functionality.

## Installation Instructions

### Step 1: Find Your JavaScript Section
Locate the `<script>` tag in your `index.html` file (near the end of the file, before `</body>`).

### Step 2: Add These Functions at the TOP of the `<script>` Section

```javascript
// ═══════════════════════════════════════════════════════════════════════════
// DATA PERSISTENCE LAYER - COPY THIS ENTIRE BLOCK TO THE TOP OF YOUR SCRIPT
// ═══════════════════════════════════════════════════════════════════════════

const STORAGE_KEYS = {
  GROUPS: 'eelChatGroups',
  WORKERS: 'eelChatWorkers',
  MESSAGES: 'eelChatMessages'
};

/**
 * Initialize localStorage if needed
 */
function initializeStorage() {
  if (!localStorage.getItem(STORAGE_KEYS.GROUPS)) {
    localStorage.setItem(STORAGE_KEYS.GROUPS, JSON.stringify({}));
  }
  if (!localStorage.getItem(STORAGE_KEYS.WORKERS)) {
    localStorage.setItem(STORAGE_KEYS.WORKERS, JSON.stringify({}));
  }
  if (!localStorage.getItem(STORAGE_KEYS.MESSAGES)) {
    localStorage.setItem(STORAGE_KEYS.MESSAGES, JSON.stringify({}));
  }
  console.log('✓ Storage initialized');
}

/**
 * Save a group chat
 * @param {string} groupId - Unique group ID
 * @param {object} groupData - { name, emoji, members, createdAt }
 */
function persistGroup(groupId, groupData) {
  try {
    const groups = JSON.parse(localStorage.getItem(STORAGE_KEYS.GROUPS)) || {};
    groups[groupId] = {
      id: groupId,
      name: groupData.name || 'Unnamed Group',
      emoji: groupData.emoji || '📦',
      members: groupData.members || [],
      createdAt: groupData.createdAt || new Date().toISOString(),
      updatedAt: new Date().toISOString()
    };
    localStorage.setItem(STORAGE_KEYS.GROUPS, JSON.stringify(groups));
    console.log('✓ Group persisted:', groupId, groupData.name);
    return true;
  } catch (e) {
    console.error('✗ Error persisting group:', e);
    return false;
  }
}

/**
 * Load all groups from storage
 * @returns {object} All groups keyed by ID
 */
function loadAllGroups() {
  try {
    const groups = JSON.parse(localStorage.getItem(STORAGE_KEYS.GROUPS)) || {};
    console.log('✓ Loaded', Object.keys(groups).length, 'group(s)');
    return groups;
  } catch (e) {
    console.error('✗ Error loading groups:', e);
    return {};
  }
}

/**
 * Load a single group by ID
 * @param {string} groupId 
 * @returns {object|null}
 */
function loadGroupById(groupId) {
  const groups = loadAllGroups();
  return groups[groupId] || null;
}

/**
 * Add a worker to a group
 * @param {string} groupId - The group ID
 * @param {string} workerId - Unique worker ID
 * @param {object} workerData - { name, role, email, phone }
 */
function persistWorkerToGroup(groupId, workerId, workerData) {
  try {
    const workers = JSON.parse(localStorage.getItem(STORAGE_KEYS.WORKERS)) || {};
    if (!workers[groupId]) {
      workers[groupId] = {};
    }
    workers[groupId][workerId] = {
      id: workerId,
      name: workerData.name || 'Unknown',
      role: workerData.role || 'Worker',
      email: workerData.email || '',
      phone: workerData.phone || '',
      status: 'active',
      addedAt: new Date().toISOString()
    };
    localStorage.setItem(STORAGE_KEYS.WORKERS, JSON.stringify(workers));
    
    // Also add to group's member list
    const groups = JSON.parse(localStorage.getItem(STORAGE_KEYS.GROUPS)) || {};
    if (groups[groupId]) {
      if (!groups[groupId].members) groups[groupId].members = [];
      if (!groups[groupId].members.includes(workerId)) {
        groups[groupId].members.push(workerId);
        localStorage.setItem(STORAGE_KEYS.GROUPS, JSON.stringify(groups));
      }
    }
    
    console.log('✓ Worker added to group:', groupId, '→', workerId);
    return true;
  } catch (e) {
    console.error('✗ Error persisting worker:', e);
    return false;
  }
}

/**
 * Load all workers in a group
 * @param {string} groupId 
 * @returns {object} Workers in the group
 */
function loadWorkersByGroup(groupId) {
  try {
    const workers = JSON.parse(localStorage.getItem(STORAGE_KEYS.WORKERS)) || {};
    return workers[groupId] || {};
  } catch (e) {
    console.error('✗ Error loading workers:', e);
    return {};
  }
}

/**
 * Remove a worker from a group
 * @param {string} groupId 
 * @param {string} workerId 
 */
function removeWorkerFromGroup(groupId, workerId) {
  try {
    const workers = JSON.parse(localStorage.getItem(STORAGE_KEYS.WORKERS)) || {};
    if (workers[groupId]) {
      delete workers[groupId][workerId];
      localStorage.setItem(STORAGE_KEYS.WORKERS, JSON.stringify(workers));
    }
    
    const groups = JSON.parse(localStorage.getItem(STORAGE_KEYS.GROUPS)) || {};
    if (groups[groupId] && groups[groupId].members) {
      groups[groupId].members = groups[groupId].members.filter(id => id !== workerId);
      localStorage.setItem(STORAGE_KEYS.GROUPS, JSON.stringify(groups));
    }
    
    console.log('✓ Worker removed from group:', groupId, '→', workerId);
    return true;
  } catch (e) {
    console.error('✗ Error removing worker:', e);
    return false;
  }
}

/**
 * Delete an entire group and all its workers
 * @param {string} groupId 
 */
function deleteGroupFromStorage(groupId) {
  try {
    const groups = JSON.parse(localStorage.getItem(STORAGE_KEYS.GROUPS)) || {};
    delete groups[groupId];
    localStorage.setItem(STORAGE_KEYS.GROUPS, JSON.stringify(groups));
    
    const workers = JSON.parse(localStorage.getItem(STORAGE_KEYS.WORKERS)) || {};
    delete workers[groupId];
    localStorage.setItem(STORAGE_KEYS.WORKERS, JSON.stringify(workers));
    
    const messages = JSON.parse(localStorage.getItem(STORAGE_KEYS.MESSAGES)) || {};
    delete messages[groupId];
    localStorage.setItem(STORAGE_KEYS.MESSAGES, JSON.stringify(messages));
    
    console.log('✓ Group deleted with all data:', groupId);
    return true;
  } catch (e) {
    console.error('✗ Error deleting group:', e);
    return false;
  }
}

/**
 * Save a message to a group
 * @param {string} groupId 
 * @param {object} messageData - { sender, text, timestamp }
 */
function persistMessage(groupId, messageData) {
  try {
    const messages = JSON.parse(localStorage.getItem(STORAGE_KEYS.MESSAGES)) || {};
    if (!messages[groupId]) {
      messages[groupId] = [];
    }
    messages[groupId].push({
      id: Date.now().toString(),
      sender: messageData.sender || 'Unknown',
      text: messageData.text || '',
      timestamp: new Date().toISOString()
    });
    localStorage.setItem(STORAGE_KEYS.MESSAGES, JSON.stringify(messages));
    return true;
  } catch (e) {
    console.error('✗ Error persisting message:', e);
    return false;
  }
}

/**
 * Load all messages for a group
 * @param {string} groupId 
 * @returns {array}
 */
function loadMessagesByGroup(groupId) {
  try {
    const messages = JSON.parse(localStorage.getItem(STORAGE_KEYS.MESSAGES)) || {};
    return messages[groupId] || [];
  } catch (e) {
    console.error('✗ Error loading messages:', e);
    return [];
  }
}

/**
 * Clear all data (useful for testing)
 */
function clearAllStorage() {
  localStorage.removeItem(STORAGE_KEYS.GROUPS);
  localStorage.removeItem(STORAGE_KEYS.WORKERS);
  localStorage.removeItem(STORAGE_KEYS.MESSAGES);
  console.log('✓ All storage cleared');
}

/**
 * Export all data as JSON (for backup)
 */
function exportData() {
  const data = {
    groups: JSON.parse(localStorage.getItem(STORAGE_KEYS.GROUPS)) || {},
    workers: JSON.parse(localStorage.getItem(STORAGE_KEYS.WORKERS)) || {},
    messages: JSON.parse(localStorage.getItem(STORAGE_KEYS.MESSAGES)) || {},
    exportedAt: new Date().toISOString()
  };
  console.log('✓ Data exported', data);
  return data;
}

// ═══════════════════════════════════════════════════════════════════════════
// END PERSISTENCE LAYER
// ═══════════════════════════════════════════════════════════════════════════
```

### Step 3: Modify Your Existing Functions

#### A. Find `openGroupModal()` function and UPDATE the SAVE button handler:

**FIND THIS:**
```javascript
function openGroupModal() {
  // ... existing code ...
  // Find where the save/create button calls a function
  // It probably looks like: btnSaveGroup.onclick = () => { ... }
}
```

**REPLACE with:**
```javascript
btnSaveGroup.onclick = () => {
  const groupName = inpGroupName.value.trim();
  const selectedEmoji = document.querySelector('.emoji-opt.sel')?.textContent || '📦';
  
  if (!groupName) {
    alert('Please enter a group name');
    return;
  }
  
  // Create unique group ID
  const groupId = 'grp_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9);
  
  // PERSIST THE GROUP
  persistGroup(groupId, {
    name: groupName,
    emoji: selectedEmoji,
    members: []
  });
  
  // Add to your UI
  addGroupToUI(groupId, groupName, selectedEmoji);
  
  // Close modal
  modalBackdrop.style.display = 'none';
  
  // Show success
  showToast('✓ Group created: ' + groupName);
};
```

#### B. Find where workers are ADDED to a group and ADD THIS:

**AFTER** the worker is added to your UI, **ADD:**
```javascript
// When you add a worker to a group:
function addWorkerToGroup(groupId, workerId, workerName, workerRole) {
  // Your existing UI code...
  
  // THEN ADD THIS:
  persistWorkerToGroup(groupId, workerId, {
    name: workerName,
    role: workerRole || 'Worker'
  });
  
  showToast('✓ Worker added: ' + workerName);
}
```

### Step 4: Initialize on App Load

**FIND** your app's initialization code (probably looks like `window.onload = function()` or `document.addEventListener('DOMContentLoaded'...)` and ADD THIS at the BEGINNING:

```javascript
// Initialize storage and load saved data
initializeStorage();

// Load and display all saved groups
const savedGroups = loadAllGroups();
Object.values(savedGroups).forEach(group => {
  addGroupToUI(group.id, group.name, group.emoji);
});

console.log('✓ App initialized with', Object.keys(savedGroups).length, 'saved group(s)');
```

### Step 5: Update Your Rendering Functions

**Update your group list rendering to use saved data:**

```javascript
function refreshGroupsList() {
  const groups = loadAllGroups();
  const sidebarList = document.getElementById('sidebar-list');
  
  sidebarList.innerHTML = '';
  
  Object.values(groups).forEach(group => {
    const chatItem = document.createElement('div');
    chatItem.className = 'chat-item';
    chatItem.innerHTML = `
      <div class="avatar group">${group.emoji}</div>
      <div class="chat-meta">
        <div class="chat-name">${group.name}</div>
        <div class="chat-preview">${group.members?.length || 0} members</div>
      </div>
    `;
    chatItem.onclick = () => selectGroup(group.id);
    sidebarList.appendChild(chatItem);
  });
}

function refreshWorkersList(groupId) {
  const workers = loadWorkersByGroup(groupId);
  const workersList = document.querySelector('.workers-list');
  
  workersList.innerHTML = '';
  
  Object.values(workers).forEach(worker => {
    const workerRow = document.createElement('div');
    workerRow.className = 'worker-row';
    workerRow.innerHTML = `
      <div class="w-av" style="background: #D4A017; color: white;">
        ${worker.name.charAt(0).toUpperCase()}
      </div>
      <div class="w-info">
        <div class="w-name">${worker.name}</div>
        <div class="w-detail">${worker.role} • Added ${new Date(worker.addedAt).toLocaleDateString()}</div>
      </div>
      <button class="mini-btn" onclick="removeWorkerFromGroup('${groupId}', '${worker.id}'); refreshWorkersList('${groupId}');">
        Remove
      </button>
    `;
    workersList.appendChild(workerRow);
  });
}
```

---

## Testing the Fix

After implementing, test in the browser console:

```javascript
// In browser DevTools Console (F12 → Console tab)

// 1. Check storage is initialized
initializeStorage();

// 2. Create a test group
persistGroup('test-1', { name: 'Test Group', emoji: '🚀', members: [] });

// 3. Add a test worker
persistWorkerToGroup('test-1', 'worker-1', { name: 'John Doe', role: 'Manager' });

// 4. Load and verify
console.log(loadAllGroups());
console.log(loadWorkersByGroup('test-1'));

// 5. Close browser, reopen app
// → Groups and workers should still be there!

// 6. Export backup
exportData();
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Data still disappears | Make sure `persistGroup()` is called AFTER group creation |
| Workers not saving | Verify `persistWorkerToGroup()` is called with correct groupId |
| Storage full error | Clear old data: `clearAllStorage()` (warning: deletes all data) |
| Functions not found | Ensure persistence code is at TOP of `<script>` section |

---

## Browser Support

✅ Works in: Chrome, Firefox, Safari, Edge (all modern versions)  
✅ Storage limit: ~5-10MB per domain  
✅ Data persists until user clears browser cache

---

**Need help? Check the console (F12) for error messages starting with `✓` (success) or `✗` (error).**
