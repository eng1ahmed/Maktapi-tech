<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  // Initial items that show when the application loads
  let videoWallItems = [
    {
      id: 'presentation',
      type: 'presentation',
      stream: null,
      videoElement: null,
      thumbnail: '/thank-you-slide.jpg',
      content: {
        title: 'Thank You!',
        subtitle: '@company.name     www.company.com     (123) 456-7890'
      }
    },
  ];

  // Add new functions for button actions
  function handleSelectAll() {
    dispatch('selectAll', { items: videoWallItems });
  }

  async function handleCloseAll() {
    // Stop all video streams before removing (except presentation)
    for (const item of videoWallItems) {
      if (item.type !== 'presentation') {  // Skip presentation items
        if (item.stream) {
          item.stream.getTracks().forEach(track => track.stop());
        }
        if (item.videoElement) {
          item.videoElement.srcObject = null;
        }
      }
    }
    
    // Keep only presentation items
    videoWallItems = videoWallItems.filter(item => item.type === 'presentation');
    dispatch('closeAll');
  }

  function handleDragOver(e) {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'copy';
  }

  async function handleDrop(e) {
    e.preventDefault();
    const sourceType = e.dataTransfer.getData('sourceType');
    const sourceId = e.dataTransfer.getData('sourceId');
    const isScreenShare = e.dataTransfer.getData('isScreenShare') === 'true';
    
    // Calculate drop position relative to video wall
    const rect = e.currentTarget.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    // Calculate grid position
    const gridColumns = Math.ceil(Math.sqrt(videoWallItems.length + 1));
    const cellWidth = rect.width / gridColumns;
    const cellHeight = rect.height / gridColumns;
    const gridX = Math.floor(x / cellWidth);
    const gridY = Math.floor(y / cellHeight);

    if (!videoWallItems.find(item => item.id === sourceId)) {
      let stream = null;

      if (sourceType === 'camera') {
        try {
          stream = await navigator.mediaDevices.getUserMedia({ 
            video: {
              width: { ideal: 1280 },
              height: { ideal: 720 }
            },
            audio: true
          });
        } catch (err) {
          console.error('Failed to get camera:', err);
        }
      } else if (isScreenShare) {
        try {
          stream = await navigator.mediaDevices.getDisplayMedia({ 
            video: {
              cursor: "always"
            },
            audio: false
          });
          
          // Handle stream ending (user stops sharing)
          stream.getVideoTracks()[0].onended = () => {
            videoWallItems = videoWallItems.filter(item => item.id !== sourceId);
          };
        } catch (err) {
          console.error('Failed to start screen sharing:', err);
        }
      }

      const newItem = {
        id: sourceId,
        type: sourceType,
        stream,
        videoElement: null,
        position: {
          x: gridX,
          y: gridY
        }
      };

      // Log the source details and total elements
      console.log('Video Wall Update:', {
        newSource: {
          id: sourceId,
          type: sourceType,
          position: {
            grid: { x: gridX, y: gridY }
          }
        },
        wallInfo: {
          totalElements: videoWallItems.length + 1,
          gridSize: {
            columns: gridColumns,
            cellWidth: Math.round(cellWidth),
            cellHeight: Math.round(cellHeight)
          }
        }
      });

      videoWallItems = [...videoWallItems, newItem];
      dispatch('sourceAdded', { sourceId, sourceType, position: newItem.position });
    }
  }

  function handleVideoRef(element, item) {
    if (element && item.stream) {
      element.srcObject = item.stream;
      element.play().catch(err => console.error('Failed to play video:', err));
    }
  }
</script>

<div class="video-wall-container">
  <div class="header">
    <div class="left">
      <div class="buttons">
        <button class="control-btn power-btn" on:click={handleCloseAll}>
          <svg viewBox="0 0 24 24" fill="currentColor" width="20" height="20">
            <path d="M13 3h-2v10h2V3zm4.83 2.17l-1.42 1.42C17.99 7.86 19 9.81 19 12c0 3.87-3.13 7-7 7s-7-3.13-7-7c0-2.19 1.01-4.14 2.58-5.42L6.17 5.17C4.23 6.82 3 9.26 3 12c0 4.97 4.03 9 9 9s9-4.03 9-9c0-2.74-1.23-5.18-3.17-6.83z"/>
          </svg>
        </button>
        <button class="control-btn success-btn" on:click={handleSelectAll}>
          <svg viewBox="0 0 24 24" fill="currentColor" width="20" height="20">
            <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zm4.59-12.42L10 14.17l-2.59-2.58L6 13l4 4 8-8z"/>
          </svg>
        </button>
      </div>
    </div>
    <div class="right">
      <span class="drag-text">Video Wall</span>
      <span class="subtitle">Drag from sources</span>
    </div>
  </div>

  <div class="video-wall" on:dragover={handleDragOver} on:drop={handleDrop}>
    <div class="video-grid" style="--grid-columns: {Math.ceil(Math.sqrt(videoWallItems.length))}">
      {#each videoWallItems as item (item.id)}
        <div 
          class="video-item"
          style="grid-column: {item.position.x + 1}; grid-row: {item.position.y + 1};"
        >
          {#if item.type === 'camera' || item.type === 'screen'}
            <video 
              autoplay 
              playsinline 
              muted
              bind:this={item.videoElement}
              use:handleVideoRef={item}
            ></video>
            {#if item.type === 'screen'}
              <div class="screen-share-overlay">
                <span class="screen-share-label">Screen Share</span>
              </div>
            {/if}
          {:else if item.type === 'presentation'}
            <div class="presentation">
              <div class="thank-you-slide">
                <h1>{item.content.title}</h1>
                <p class="contact-info">{item.content.subtitle}</p>
              </div>
            </div>
          {:else if item.type === 'conference'}
            <div class="conference-grid">
              <div class="participants-grid">
                {#each Array(item.content.participants) as _, i}
                  <div class="participant-cell"></div>
                {/each}
              </div>
            </div>
          {:else if item.type === 'dashboard'}
            <div class="dashboard">
              <img src={item.thumbnail} alt="Analytics Dashboard" />
            </div>
          {:else if item.type === 'sharepoint'}
            <div class="sharepoint-item">
              {#if item.content?.type === 'document'}
                <div class="document-preview">
                  <div class="document-icon">
                    {#if item.content.fileType?.includes('pdf')}
                      <svg viewBox="0 0 24 24" fill="currentColor" width="48" height="48">
                        <path d="M20 2H8c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2zm-8.5 7.5c0 .83-.67 1.5-1.5 1.5H9v2H7.5V7H10c.83 0 1.5.67 1.5 1.5v1zm5 2c0 .83-.67 1.5-1.5 1.5h-2.5V7H15c.83 0 1.5.67 1.5 1.5v3zm4-3H19v1h1.5V11H19v2h-1.5V7h3v1.5zM9 9.5h1v-1H9v1zM4 6H2v14c0 1.1.9 2 2 2h14v-2H4V6zm10 5.5h1v-3h-1v3z"/>
                      </svg>
                    {:else if item.content.fileType?.includes('excel') || item.content.fileType?.includes('xlsx')}
                      <svg viewBox="0 0 24 24" fill="#217346" width="48" height="48">
                        <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-3.5 14H14v-2.5h-4V17H8.5v-2.5h4V12H14v2.5h1.5V17zm-6-9h1.5v1.5H13V7h1.5v1.5H16V10h-1.5v1.5H13V10h-1.5v1.5H10V10H8.5V8.5H10V7z"/>
                      </svg>
                    {:else if item.content.fileType?.includes('word') || item.content.fileType?.includes('docx')}
                      <svg viewBox="0 0 24 24" fill="#2B579A" width="48" height="48">
                        <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-3 10H8v-2h8v2zm0-4H8V7h8v2z"/>
                      </svg>
                    {:else}
                      <svg viewBox="0 0 24 24" fill="currentColor" width="48" height="48">
                        <path d="M14 2H6c-1.1 0-1.99.9-1.99 2L4 20c0 1.1.89 2 1.99 2H18c1.1 0 2-.9 2-2V8l-6-6zm2 16H8v-2h8v2zm0-4H8v-2h8v2zm-3-5V3.5L18.5 9H13z"/>
                      </svg>
                    {/if}
                  </div>
                  <div class="document-info">
                    <h3>{item.content.name || 'Unknown Document'}</h3>
                    <p>{item.content.modified || 'Unknown date'}</p>
                    {#if item.content.fileType}
                      <span class="file-type">{item.content.fileType.toUpperCase()}</span>
                    {/if}
                  </div>
                </div>
              {:else if item.content?.type === 'folder'}
                <div class="folder-preview">
                  <div class="folder-icon">
                    <svg viewBox="0 0 24 24" fill="currentColor" width="48" height="48">
                      <path d="M10 4H4c-1.1 0-1.99.9-1.99 2L2 18c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V8c0-1.1-.9-2-2-2h-8l-2-2z"/>
                    </svg>
                  </div>
                  <div class="folder-info">
                    <h3>{item.content.name || 'Unknown Folder'}</h3>
                    <p>{item.content.itemCount || 0} items</p>
                  </div>
                </div>
              {:else}
                <div class="document-preview">
                  <div class="document-icon">
                    <svg viewBox="0 0 24 24" fill="currentColor" width="48" height="48">
                      <path d="M14 2H6c-1.1 0-1.99.9-1.99 2L4 20c0 1.1.89 2 1.99 2H18c1.1 0 2-.9 2-2V8l-6-6zm2 16H8v-2h8v2zm0-4H8v-2h8v2zm-3-5V3.5L18.5 9H13z"/>
                    </svg>
                  </div>
                  <div class="document-info">
                    <h3>Unknown Item</h3>
                    <p>No details available</p>
                  </div>
                </div>
              {/if}
            </div>
          {/if}
        </div>
      {/each}
    </div>
  </div>
</div>

<style>
  .video-wall-container {
    width: 100%;
    height: 100%;
    background-color: white;
    border-radius: 12px;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  }

  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 1.5rem;
    background-color: white;
    border-bottom: 1px solid #eee;
  }

  .left {
    display: flex;
    align-items: center;
    gap: 1rem;
  }

  h2 {
    margin: 0;
    font-size: 1.25rem;
    color: #333;
    font-weight: 500;
  }

  .buttons {
    display: flex;
    gap: 0.5rem;
  }

  .control-btn {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    border: none;
    background: transparent;
    color: #666;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;
  }

  .control-btn:hover {
    background-color: #f5f5f5;
  }

  .power-btn {
    color: #ff3e3e;
  }

  .power-btn:hover {
    background-color: rgba(255, 62, 62, 0.1);
    color: #ff3e3e;
  }

  .success-btn {
    color: #4CAF50;
  }

  .success-btn:hover {
    background-color: rgba(76, 175, 80, 0.1);
    color: #4CAF50;
  }

  .right {
    display: flex;
    flex-direction: column;
    align-items: flex-end;
  }

  .drag-text {
    color: #333;
    font-size: 1rem;
    font-weight: 500;
  }

  .subtitle {
    color: #999;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }

  .video-wall {
    flex: 1;
    padding: 1rem;
    background-color: white;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 0;
    overflow: auto;
  }

  .video-grid {
    display: grid;
    grid-template-columns: repeat(var(--grid-columns), 1fr);
    grid-auto-rows: 1fr;
    gap: 1rem;
    width: 100%;
    height: 100%;
    min-height: 0;
  }

  .video-item {
    position: relative;
    aspect-ratio: 16/9;
    background-color: #f8f8f8;
    border-radius: 8px;
    overflow: hidden;
    border: 1px solid #eee;
    transition: all 0.3s ease;
  }

  .video-item video {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .presentation {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    background-color: #000;
  }

  .presentation img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
  }

  .thank-you-slide {
    width: 100%;
    height: 100%;
    background-color: #000;
    color: white;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    padding: 2rem;
    text-align: center;
  }

  .thank-you-slide h1 {
    font-size: 3rem;
    margin: 0 0 1rem 0;
  }

  .thank-you-slide .contact-info {
    font-size: 0.9rem;
    color: #999;
    margin: 0;
  }

  .conference-grid {
    width: 100%;
    height: 100%;
    background-color: #1a1a1a;
    padding: 0.5rem;
  }

  .participants-grid {
    width: 100%;
    height: 100%;
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    grid-template-rows: repeat(3, 1fr);
    gap: 0.5rem;
  }

  .participant-cell {
    background-color: #2a2a2a;
    border-radius: 4px;
    width: 100%;
    height: 100%;
  }

  .dashboard {
    width: 100%;
    height: 100%;
    background-color: white;
    padding: 1rem;
  }

  .dashboard img {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .sharepoint-item {
    width: 100%;
    height: 100%;
    background-color: white;
    padding: 1.5rem;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid #eee;
    border-radius: 8px;
    transition: all 0.2s ease;
  }

  .sharepoint-item:hover {
    border-color: #0078d4;
    box-shadow: 0 2px 8px rgba(0, 120, 212, 0.1);
  }

  .document-preview, .folder-preview {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }

  .document-icon, .folder-icon {
    color: #0078d4;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .folder-icon {
    color: #ffd700;
  }

  .document-info, .folder-info {
    text-align: center;
  }

  .document-info h3, .folder-info h3 {
    margin: 0;
    font-size: 1rem;
    color: #333;
    font-weight: 500;
    margin-bottom: 0.25rem;
  }

  .document-info p, .folder-info p {
    margin: 0;
    font-size: 0.875rem;
    color: #666;
  }

  .file-type {
    display: inline-block;
    padding: 2px 6px;
    background-color: #f0f0f0;
    border-radius: 4px;
    font-size: 0.75rem;
    color: #666;
    margin-top: 0.5rem;
  }

  .document-icon svg {
    filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.1));
  }

  .screen-share-overlay {
    position: absolute;
    top: 0.5rem;
    left: 0.5rem;
    background-color: rgba(0, 0, 0, 0.6);
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
  }

  .screen-share-label {
    color: white;
    font-size: 0.75rem;
    font-weight: 500;
  }

  .video-item:hover {
    border-color: #0078d4;
    box-shadow: 0 2px 8px rgba(0, 120, 212, 0.1);
  }
</style> 