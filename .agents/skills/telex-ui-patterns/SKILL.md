---
name: telex-ui-patterns
description: WordPress block editor UI patterns from telex-podcast-player for Eclecty media plugins. Use when developing new blocks or refactoring existing ones that require advanced inspector controls like inline-editable arrays (playlists, timestamps), button group toggles, or in-block RichText editing. Or when user mentions "telex patterns", "media player ui", "playlist UI", or "block.json attributes for arrays".
---

# Telex UI Patterns for Block Editor

## Overview

This skill documents four proven UI patterns extracted from the `telex-podcast-player` block. These patterns are intended to be reused across other Eclecty plugins (such as `eclecty-media-player`, `eclecty-video-playlist`, etc.) to ensure a consistent, high-quality block editor experience.

## When to Use

**Use when:**
- Building out inspector controls for media metadata (chapters, playlists).
- Needing to represent array data compactly in the WordPress sidebar (`InspectorControls > PanelBody`).
- Replacing standard `<SelectControl>` dropdowns with more visual `<ButtonGroup>` toggles for visual layout modes or skins.
- Implementing in-block WYSIWYG editing for complex metadata beyond basic paragraphs (e.g. podcast titles, descriptions).

## 4 Core UI Patterns

### 1. ButtonGroup Toggles for Mode/Layout

Instead of a dropdown, use a `ButtonGroup` to toggle between a small number of mutually exclusive states (like a light/dark mode or a horizontal/vertical layout). This provides a single-click experience.

**Implementation Pattern:**

```javascript
import { __ } from '@wordpress/i18n';
import { PanelBody, ButtonGroup, Button } from '@wordpress/components';
import { InspectorControls } from '@wordpress/block-editor';

// Inside your Edit function:
<InspectorControls>
	<PanelBody title={ __( 'Skin', 'text-domain' ) }>
		<ButtonGroup style={ { display: 'flex', width: '100%' } }>
			<Button
				variant={ skin === 'light' ? 'primary' : 'secondary' }
				onClick={ () => setAttributes( { skin: 'light' } ) }
				style={ { flex: 1, justifyContent: 'center' } }
			>
				{ __( 'Light', 'text-domain' ) }
			</Button>
			<Button
				variant={ skin === 'dark' ? 'primary' : 'secondary' }
				onClick={ () => setAttributes( { skin: 'dark' } ) }
				style={ { flex: 1, justifyContent: 'center' } }
			>
				{ __( 'Dark', 'text-domain' ) }
			</Button>
		</ButtonGroup>
	</PanelBody>
</InspectorControls>
```

**Key Takeaways:**
- Set the `ButtonGroup` width to `100%` with `display: flex`.
- Set each `Button` flex basis to `1` so they share horizontal space evenly.
- Use `variant="primary"` for the active state and `variant="secondary"` for the inactive state.

---

### 2. Inline-editable Playlist (Arrays with custom fields, reordering, deletion)

Complex array data shouldn't rely solely on modal popups. Render the array directly in the inspector sidebar, mapped to editable text fields and utility buttons for rearranging items.

**Implementation Pattern:**

1.  **State Schema (`block.json`)**: Ensure the attribute is typed as `array`.
    ```json
    "playlist": {
        "type": "array",
        "default": []
    }
    ```
2.  **State Management Functions**:
    ```javascript
    // Remove Item
    const onRemoveTrack = ( index ) => {
        const updated = [ ...playlist ];
        updated.splice( index, 1 );
        setAttributes( { playlist: updated } );
    };

    // Reorder Item
    const onMoveTrack = ( index, direction ) => {
        const updated = [ ...playlist ];
        const newIndex = index + direction;
        if ( newIndex < 0 || newIndex >= updated.length ) return;
        const temp = updated[ index ];
        updated[ index ] = updated[ newIndex ];
        updated[ newIndex ] = temp;
        setAttributes( { playlist: updated } );
    };

    // Update String Field in Object
    const onUpdateTrackTitle = ( index, title ) => {
        const updated = [ ...playlist ];
        updated[ index ] = { ...updated[ index ], title };
        setAttributes( { playlist: updated } );
    };
    ```

3.  **UI Render** (Inside a `PanelBody` in `InspectorControls`):
    ```javascript
    import { Button, TextControl } from '@wordpress/components';

    <div className="inspector-playlist">
        { playlist.map( ( track, index ) => (
            <div key={ index } className="inspector-playlist__item">
                <div className="item-header">
                    <span>{ index + 1 }. { track.filename }</span>
                </div>
                <div className="item-controls" style={ { display: 'flex', gap: '4px', alignItems: 'center' } }>
                    <Button
                        size="small"
                        icon="arrow-up-alt2"
                        onClick={ () => onMoveTrack( index, -1 ) }
                        disabled={ index === 0 }
                    />
                    <Button
                        size="small"
                        icon="arrow-down-alt2"
                        onClick={ () => onMoveTrack( index, 1 ) }
                        disabled={ index === playlist.length - 1 }
                    />
                    <TextControl
                        value={ track.title || '' }
                        onChange={ ( val ) => onUpdateTrackTitle( index, val ) }
                        placeholder={ __( 'Track title...', 'text-domain' ) }
                        __nextHasNoMarginBottom
                        style={ { flex: 1 } }
                    />
                    <Button
                        size="small"
                        icon="trash"
                        isDestructive
                        onClick={ () => onRemoveTrack( index ) }
                    />
                </div>
            </div>
        ) ) }
    </div>
    ```

---

### 3. Inline-editable Timestamps / Chapters

Similar to playlists, but dealing with time-formatting concerns. User input of "1:30" string must be parsed into integer/float seconds for the database storage, then reformatted when displaying the input again.

**Implementation Pattern:**

1.  **Helper Functions**:
    ```javascript
    function formatTimeFromSeconds( totalSeconds ) {
        if ( ! totalSeconds ) return '0:00';
        var s = Math.floor( totalSeconds );
        var h = Math.floor( s / 3600 );
        var m = Math.floor( ( s % 3600 ) / 60 );
        var sec = s % 60;
        if ( h > 0 ) {
            return h + ':' + ( m < 10 ? '0' : '' ) + m + ':' + ( sec < 10 ? '0' : '' ) + sec;
        }
        return m + ':' + ( sec < 10 ? '0' : '' ) + sec;
    }

    function parseTimeString( str ) {
        if ( ! str ) return 0;
        var parts = str.split( ':' ).map( Number );
        if ( parts.length === 3 ) {
            return ( parts[ 0 ] * 3600 ) + ( parts[ 1 ] * 60 ) + ( parts[ 2 ] || 0 );
        }
        if ( parts.length === 2 ) {
            return ( parts[ 0 ] * 60 ) + ( parts[ 1 ] || 0 );
        }
        return parseInt( str, 10 ) || 0;
    }
    ```

2.  **Updating State**:
    ```javascript
    const onUpdateTimestamp = ( index, field, value ) => {
        const updated = [ ...currentTimestamps ];
        if ( field === 'timeStr' ) {
            // Convert string to seconds immediately on update
            updated[ index ] = { ...updated[ index ], time: parseTimeString( value ) };
        } else {
            updated[ index ] = { ...updated[ index ], [ field ]: value };
        }
        setAttributes( { timestamps: updated } );
    };
    ```

3.  **UI Render** (Inside `TextControl`):
    ```javascript
    <TextControl
        value={ formatTimeFromSeconds( ts.time ) }
        onChange={ ( val ) => onUpdateTimestamp( index, 'timeStr', val ) }
        placeholder="0:00"
        __nextHasNoMarginBottom
    />
    ```

---

### 4. In-block RichText Fields

Instead of hiding critical text metadata (like podcast cast name or episode title) entirely in the sidebar sidebar or requiring custom blocks, make those text strings editable right in the block preview area.

**Implementation Pattern:**

1.  **State Schema (`block.json`)**: Ensure attribute type matches expectations.
    ```json
    "episodeTitle": {
        "type": "string"
    },
    "episodeDescription": {
        "type": "string"
    }
    ```

2.  **UI Render** (Inside your `Edit` JSX block layout area):
    ```javascript
    import { RichText } from '@wordpress/block-editor';

    <div className="player__info">
        <RichText
            tagName="span" // Render as inline span
            className="player__title"
            placeholder={ __( 'Podcast Name | Season | Episode Title', 'text-domain' ) }
            value={ episodeTitle }
            onChange={ ( value ) => setAttributes( { episodeTitle: value } ) }
            allowedFormats={ [] } // Strip all bold/italic formatting if strictly plaintext
        />
        <RichText
            tagName="p" // Render as paragraph
            className="player__description"
            placeholder={ __( 'Episode description...', 'text-domain' ) }
            value={ episodeDescription }
            onChange={ ( value ) => setAttributes( { episodeDescription: value } ) }
            allowedFormats={ [ 'core/bold', 'core/italic' ] } // Enable basic formats
        />
    </div>
    ```

**Key Takeaways:**
- If the text should only be unformatted string data, set `allowedFormats={ [] }`. This prevents users pasting rich HTML where you don't want it.
- Ensure the `tagName` makes sense for the layout (often `span`, `h3`, or `p`).
- Give it a helpful descriptive placeholder string.
