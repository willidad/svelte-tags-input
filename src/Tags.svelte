<script>
import { createEventDispatcher } from 'svelte';

const dispatch = createEventDispatcher();

let tag = "";
let arrelementsmatch = [];
let regExpEscape = (s) => {
  return s.replace(/[-\\^$*+?.()|[\]{}]/g, "\\$&")
}

export let tags;
export let addKeys;
export let maxTags;
export let onlyUnique;
export let removeKeys;
export let placeholder;
export let allowPaste;
export let allowDrop;
export let splitWith;
export let autoComplete;
export let autoCompleteFilter;
export let autoCompleteKey;
export let autoCompleteMarkupKey;
export let name;
export let id;
export let allowBlur;
export let disable;
export let minChars;
export let onlyAutocomplete;
export let label;
export let message;
export let error;
export let messagePersist;


let layoutElement;
let dirty = false;

$: tags = tags || [];
$: addKeys = addKeys || [13];
$: maxTags = maxTags || false;
$: onlyUnique = onlyUnique || false;
$: removeKeys = removeKeys || [8];
$: placeholder = placeholder || "";
$: allowPaste = allowPaste || false;
$: allowDrop = allowDrop || false;
$: splitWith = splitWith || ",";
$: autoComplete = autoComplete || false;
$: autoCompleteFilter = typeof autoCompleteFilter == "undefined" ? true : false;
$: autoCompleteKey = autoCompleteKey || false;
$: autoCompleteMarkupKey = autoCompleteMarkupKey || false;
$: name = name || "svelte-tags-input";
$: id = id || uniqueID();
$: allowBlur = allowBlur || false;
$: disable = disable || false;
$: minChars = minChars || 1;
$: onlyAutocomplete = onlyAutocomplete || false;
$: label = label || name;
$: labelShow = labelShow || false;
$: dirty = tags.length > 0 || arrelementsmatch.length > 0 //ensure that label stays i place when imput looses focus when navigation to autocompletelist

$: matchsID = id + "_matchs";

let storePlaceholder = placeholder;

function setTag(e) {
    const currentTag = e.target.value;
    
    if (addKeys) {
        addKeys.forEach(function(key) {
            if (key === e.keyCode) {
                
                if (currentTag) e.preventDefault();

                if (autoComplete && arrelementsmatch.length > 0) {
                    addTag(arrelementsmatch[0].label);
                } else {
                    addTag(currentTag);
                }
            }
        });
    }
    
    if (removeKeys) {
        removeKeys.forEach(function(key) {
            if (key === e.keyCode && tag === "") {
                removeTag(tags.length-1) // thie last tag
            }
        });
    }
    
    // ArrowDown : focus on first element of the autocomplete
    if (e.keyCode === 40 && autoComplete && document.getElementById(matchsID)) {
        e.preventDefault();
        document.getElementById(matchsID).querySelector("li:first-child").focus();
    } // ArrowUp : focus on last element of the autocomplete
    else if (e.keyCode === 38 && autoComplete && document.getElementById(matchsID)) {
        e.preventDefault();
        document.getElementById(matchsID).querySelector("li:last-child").focus();
    }

}

function addTag(currentTag) {

    if (typeof currentTag === 'object' && currentTag !== null) {
        if (!autoCompleteKey) {
            return console.error("'autoCompleteKey' is necessary if 'autoComplete' result is an array of objects");
        }

        var currentObjTags = currentTag;
        currentTag = currentTag[autoCompleteKey].trim();

    } else {
        currentTag = currentTag.trim();
    }
    
    if (currentTag == "") return;
    if (maxTags && tags.length == maxTags) return;
    if (onlyUnique && tags.includes(currentTag)) return;
    if (onlyAutocomplete && arrelementsmatch.length === 0) return;

    let newTag = currentObjTags ? currentObjTags : currentTag
    // do not mute the tags array with push or pop. 
    tags = tags.concat([newTag])
    tag = "";

    dispatch('tagAdded', {
        tag: newTag
    });
    
    // Hide autocomplete list
    // Focus on svelte tags input
    arrelementsmatch = [];
    document.getElementById(id).focus();

    if (maxTags && tags.length == maxTags) {
        document.getElementById(id).readOnly = true;
        placeholder = "";
    };

}

function removeTag(i) {
    // do not mute the tags array with push or pop. 
    let removedTag = tags[i];
    tags = tags.slice(0, i).concat(tags.slice(i+1));

    dispatch('tagRemoved', {
		tag: removedTag
	});    
    
    // Hide autocomplete list
    // Focus on svelte tags input
    arrelementsmatch = [];
    document.getElementById(id).readOnly = false;
    placeholder = storePlaceholder;
    document.getElementById(id).focus();
}

function onPaste(e) {

    if(!allowPaste) return;
    e.preventDefault();

    const data = getClipboardData(e);
    splitTags(data).map(tag => addTag(tag));    
}

function onDrop(e) {

    if(!allowDrop) return;
    e.preventDefault();

    const data = e.dataTransfer.getData("Text");
    splitTags(data).map(tag => addTag(tag));
}

function onFocus() {
    layoutElement.classList.add('focus');
}

function onBlur(e, tag) {
    layoutElement.classList.remove('focus');

    if (!document.getElementById(matchsID) && allowBlur) {
        e.preventDefault();
        addTag(tag);
    }
}

function onClick() {    
    minChars == 0 && getMatchElements();
}


function getClipboardData(e) {

    if (window.clipboardData) {
        return window.clipboardData.getData('Text')
    }

    if (e.clipboardData) {
        return e.clipboardData.getData('text/plain')
    }

    return ''
}

function splitTags(data) {
    return data.split(splitWith).map(tag => tag.trim());    
}

function escapeHTML(string) {
    const htmlEscapes = {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#x27;',
        '/': '&#x2F;'
    };
    return ('' + string).replace(/[&<>"'\/]/g, match => htmlEscapes[match]);
}

function buildMatchMarkup(search, value) {
    return escapeHTML(value).replace(RegExp(regExpEscape(search.toLowerCase()), 'i'), "<strong>$&</strong>")
}

async function getMatchElements(input) {

    if (!autoComplete) return;
    
    let value = input ? input.target.value : "";
    let autoCompleteValues = [];
    
    if (Array.isArray(autoComplete)) {
        autoCompleteValues = autoComplete
    }
            
    if (typeof autoComplete === 'function') {
        if(autoComplete.constructor.name === 'AsyncFunction') {
            autoCompleteValues = await autoComplete(value)
        } else {
            autoCompleteValues = autoComplete(value)
        }
    }

    if(autoCompleteValues.constructor.name === 'Promise') {
        autoCompleteValues = await autoCompleteValues;
    }
    
    // Escape
    if ((minChars > 0 && value == "") || (input && input.keyCode === 27) || value.length < minChars ) {
        arrelementsmatch = [];
        return;
    }

    let matchs = autoCompleteValues;
    
    if (typeof autoCompleteValues[0] === 'object' && autoCompleteValues !== null) {
        
        if (!autoCompleteKey) {
            return console.error("'autoCompleteValue' is necessary if 'autoComplete' result is an array of objects");
        }

        if(autoCompleteFilter !== false) {
            matchs = autoCompleteValues.filter(e => e[autoCompleteKey].toLowerCase().includes(value.toLowerCase()))
        }
        matchs = matchs.map(matchTag => {
            return {
                label: matchTag,
                search: autoCompleteMarkupKey ? matchTag[autoCompleteMarkupKey] : buildMatchMarkup(value, matchTag[autoCompleteKey])
            }
        });


    } else {
        if(autoCompleteFilter !== false) {
            matchs = autoCompleteValues.filter(e => e.toLowerCase().includes(value.toLowerCase()))
        }
        matchs = matchs.map(matchTag => {
            return {
                label: matchTag,
                search: buildMatchMarkup(value, matchTag)
            }
        });
    }

    if (onlyUnique === true && !autoCompleteKey) {
        matchs = matchs.filter(tag => !tags.includes(tag.label));
    }

    arrelementsmatch = matchs;
}

function navigateAutoComplete(e, autoCompleteIndex, autoCompleteLength, autoCompleteElement) {

    if (!autoComplete) return;
    
    e.preventDefault();

    // ArrowDown
    if (e.keyCode === 40) {
        // Last element on the list ? Go to the first
        if (autoCompleteIndex + 1 === autoCompleteLength) {
            document.getElementById(matchsID).querySelector("li:first-child").focus();
            return;
        }
        document.getElementById(matchsID).querySelectorAll("li")[autoCompleteIndex + 1].focus();
    } else if (e.keyCode === 38) {
        // ArrowUp
        // First element on the list ? Go to the last
        if (autoCompleteIndex === 0) {
            document.getElementById(matchsID).querySelector("li:last-child").focus();
            return;
        }
        document.getElementById(matchsID).querySelectorAll("li")[autoCompleteIndex - 1].focus();
    } else if (e.keyCode === 13) { 
        // Enter
        addTag(autoCompleteElement);
    } else if (e.keyCode === 27) {
        // Escape
        arrelementsmatch = [];
        document.getElementById(id).focus();
    }
}

function uniqueID() {
    return 'sti_' + Math.random().toString(36).substring(2, 11);
};

</script>
<div class="textfield" class:dirty>
    <div class="tags-container" class:sti-layout-disable={disable} bind:this={layoutElement}>
        {#if tags.length > 0}
            {#each tags as tag, i}
                <span class="input-tag">
                    {#if typeof tag === 'string'}
                        {tag}
                    {:else}
                        {tag[autoCompleteKey]}
                    {/if}
                    {#if !disable}
                    <span class="input-tag-remove" on:click={() => removeTag(i)}> &#215;</span>
                    {/if}
                </span>
            {/each}
        {/if}
        <input
            type="text"
            id={id}
            name={name}
            bind:value={tag}
            on:keydown={setTag}
            on:keyup={getMatchElements}
            on:paste={onPaste}
            on:drop={onDrop}
            on:focus={onFocus}
            on:blur={(e) => onBlur(e, tag)}
            on:click={onClick}
            class="tags-input"
            placeholder={placeholder}
            disabled={disable}
        >
        <div class="label">
            {label}
          </div>
        <div class="focus-line"></div>
        <div class="input-line"></div>
        {#if !!message || !!error}
            <div class="help" class:persist={messagePersist} class:error>
            <div class="message">{error || message}</div>
            </div>
        {/if}
    </div>
    
</div>

{#if autoComplete && arrelementsmatch.length > 0}
    <div class="tags-input-matchs-parent">
        <ul id="{id}_matchs" class="tags-input-matchs">
            {#each arrelementsmatch as element, index}
                <li
                    tabindex="-1"
                    on:keydown={(e) => navigateAutoComplete(e, index, arrelementsmatch.length, element.label)}
                    on:click={() => addTag(element.label)}>
                        {@html element.search}
                    </li>
            {/each}
        </ul>
    </div>
{/if}

<style>
/* CSS svelte-tags-input */

.textfield {
    font-family: Roboto, "Segoe UI", sans-serif;
    font-weight: 400;
    font-size: inherit;
    text-decoration: inherit;
    text-transform: inherit;
    box-sizing: border-box;
    margin: 0 0 20px;
    position: relative;
    width: 100%;
    background-color: inherit;
    will-change: opacity, transform, color;
    min-height: 53px;
}

.input-line {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    margin: 0;
    height: 1px;
    background: rgba(0, 0, 0, 0.3755);
    /* postcss-custom-properties: ignore next */
    background: var(--label, rgba(0, 0, 0, 0.3755));
  }

.focus-line {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 2px;
    -webkit-transform: scaleX(0);
    /* autoprefixer: ignore next */
    transform: scaleX(0);
    /* autoprefixer: ignore next */
    transition: transform 0.18s cubic-bezier(0.4, 0, 0.2, 1), opacity 0.18s cubic-bezier(0.4, 0, 0.2, 1),
        -webkit-transform 0.18s cubic-bezier(0.4, 0, 0.2, 1);
    /* autoprefixer: ignore next */
    transition: transform 0.18s cubic-bezier(0.4, 0, 0.2, 1), opacity 0.18s cubic-bezier(0.4, 0, 0.2, 1);
    opacity: 0;
    z-index: 2;
    background: #1976d2;
    /* postcss-custom-properties: ignore next */
    background: var(--primary, #1976d2);
}


.baseline {
    
}

/*.tags-input,
.input-tag,
.tags-input-matchs,
.label {
    font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Oxygen,Ubuntu,Cantarell,"Fira Sans","Droid Sans","Helvetica Neue",sans-serif;
    font-size: 14px;
    padding: 5px 10px;
}*/

.label {
    font: inherit;
    display: inline-flex;
    position: absolute;
    left: 0;
    top: 28px;
    padding-right: 0.2em;
    color: rgba(0, 0, 0, 0.3755);
    /* postcss-custom-properties: ignore next */
    color: var(--label, rgba(0, 0, 0, 0.3755));
    background-color: inherit;
    pointer-events: none;
    -webkit-backface-visibility: hidden;
    backface-visibility: hidden;
    overflow: hidden;
    max-width: 90%;
    white-space: nowrap;
    transform-origin: left top;
    transition: 0.18s cubic-bezier(0.25, 0.8, 0.5, 1);
  }

/* svelte-tags-input-layout */

.tags-container {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -ms-flex-wrap: wrap;
    flex-wrap: wrap;
    -webkit-box-align: center;
    -ms-flex-align: center;
    background: #FFF;
    border-radius: 2px;
    align-items: center;
    padding-top: 25px;
    padding-bottom: 3px;
}
.tags-container:hover .input-line  {
    background: #333;
    /* postcss-custom-properties: ignore next */
    background: var(--color, #333);
}



/*.tags-container:focus,
.tags-container:hover {
    border: solid 1px #000;    
}*/

/* svelte-tags-input */

.tags-input {
    -webkit-box-flex: 1;
    -ms-flex: 1;
    flex: 1;
    margin: 5px 5px 5px 0;
    border: none;
}

.tags-input:focus {
    outline:0;
}
.tags-input:focus ~ .focus-line {
    transform: scaleX(1);
    opacity: 1;
  }

.dirty .label {
    letter-spacing: 0.4px;
    top: 6px;
    bottom: unset;
    font-size: 13px;
  }
.tags-input:focus ~ .label {
    letter-spacing: 0.4px;
    top: 6px;
    bottom: unset;
    font-size: 13px;
    color: #1976d2;
    /* postcss-custom-properties: ignore next */
    color: var(--primary, #1976d2);
  }

/* svelte-tags-input-tag */

.input-tag {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    white-space: nowrap;
    list-style: none;
    background: #333;
    color: #FFF;
    border-radius: 1rem;
    margin: 3px 5px 3px 0;
    padding: 0 10px;
    ;
}

/*.svelte-tags-input-tag:hover {
    background: #CCC;
}*/

.input-tag-remove {
    cursor:pointer;
    padding-left: 2px;
}

/* svelte-tags-input-matchs */

.tags-input-matchs-parent {
    position:relative;
}

.tags-input-matchs {
    position:absolute;
    top:-20px;
    left:0;
    right:0;
    margin:3px 0;
    padding: 0px;
    background:#FFF;
    border: solid 1px #CCC;
    border-radius: 2px;
    max-height:310px;
    overflow-x:auto;
    z-index: 100;
}

.tags-input-matchs li {
    list-style:none;
    padding:5px;
    border-radius: 2px;
    cursor:pointer;
}

.tags-input-matchs li:hover,
.tags-input-matchs li:focus {
    background:#000;
    color:#FFF;
    outline:none;
}


.tags-container.sti-layout-disable,
.input-tag:disabled {
    /*background: #EAEAEA;*/
    cursor: not-allowed;
    opacity: 0.5;
}

.help {
    position: absolute;
    left: 0;
    right: 0;
    bottom: -18px;
    display: flex;
    justify-content: space-between;
    font-size: 12px;
    line-height: normal;
    letter-spacing: 0.4px;
    color: rgba(0, 0, 0, 0.3755);
    /* postcss-custom-properties: ignore next */
    color: var(--label, rgba(0, 0, 0, 0.3755));
    opacity: 0;
    overflow: hidden;
    max-width: 90%;
    white-space: nowrap;
  }
  .persist,
  .error,
  .tags-input:focus ~ .help {
    opacity: 1;
  }
  .error {
    color: #ff5252;
  }

/*.tags-container.sti-layout-disable:hover,
.tags-container.sti-layout-disable:focus {
    border-color:rgb(117, 117, 117);
}

/*.tags-input-layout.sti-layout-disable .tags-input-tag {
    /*background: #333;
    opacity: 0.5;
}

.tags-container.sti-layout-disable .tags-input-remove {
    cursor: not-allowed;
}

.tags-container label {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}*/
</style>