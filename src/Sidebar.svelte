<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  // Active menu item tracking
  let activeItem = 'home';
  
  const menuItems = [
    { id: 'home', icon: '🏠', label: 'Home' },
    { id: 'display', icon: '🖥️', label: 'Display' },
    { id: 'splitScreen', icon: '⚡', label: 'Split Screen' },
    { id: 'camera', icon: '📹', label: 'Camera' },
    { id: 'settings', icon: '⚙️', label: 'Settings' }
  ];

  function handleMenuClick(id) {
    activeItem = id;
    dispatch('menuSelect', { id });
  }
</script>

<div class="sidebar">
  <div class="logo">
    <img src="your-logo.png" alt="Logo" />
  </div>

  <nav class="menu">
    {#each menuItems as item}
      <button
        class="menu-item"
        class:active={activeItem === item.id}
        on:click={() => handleMenuClick(item.id)}
      >
        <span class="icon">{item.icon}</span>
      </button>
    {/each}
  </nav>

  <div class="bottom-menu">
    <button class="menu-item">
      <span class="icon">⚪</span>
    </button>
    <button class="menu-item">
      <span class="icon">🚪</span>
    </button>
  </div>
</div>

<style>
  .sidebar {
    background-color: #1a1a1a;
    width: 72px;
    height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 1rem 0;
    box-sizing: border-box;
  }

  .logo {
    margin-bottom: 2rem;
    width: 40px;
    height: 40px;
  }

  .logo img {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .menu {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    width: 100%;
  }

  .menu-item {
    width: 100%;
    padding: 0.75rem 0;
    background: transparent;
    border: none;
    color: #808080;
    cursor: pointer;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: all 0.2s ease;
  }

  .menu-item:hover {
    background-color: rgba(255, 255, 255, 0.1);
    color: #ffffff;
  }

  .menu-item.active {
    color: #4CAF50;
    background-color: rgba(76, 175, 80, 0.1);
  }

  .icon {
    font-size: 1.25rem;
  }

  .bottom-menu {
    margin-top: auto;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    width: 100%;
  }
</style> 