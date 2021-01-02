<template>
  <div class="hello">
    <h1>vue-a11y-autocomplete</h1>
    <div class="container">
      <label for="destination">
        <span>Destination</span>
      </label>
      <div class="autocomplete">
        <input
          aria-owns="autocomplete-options--destination"
          autocapitalize="none"
          type="text"
          autocomplete="off"
          aria-autocomplete="list"
          role="combobox"
          id="destination"
          :aria-expanded="menuOpen ? 'true' : 'false'"
          ref="destination"
          @keyup="onTextBoxKeyUp"
          @keydown="onTextBoxKeyDown"
          @click="onTextBoxClick"
          @focus="onTextBoxFocus"
          v-model="inputValue"
          :class="{ 'autocomplete-isFocused': isFocused }"
        />
        <svg
          focusable="false"
          version="1.1"
          xmlns="http://www.w3.org/2000/svg"
          @click="onArrowClick"
        >
          <g><polygon points="0 0 22 0 11 17"></polygon></g>
        </svg>
        <ul
          id="autocomplete-options--destination"
          role="listbox"
          :class="{ hidden: !menuOpen }"
          @keydown="onMenuKeyDown"
        >
          <template v-if="results.length">
            <li
              v-for="(option, i) in results"
              :key="option.code"
              role="option"
              tabindex="-1"
              :id="`autocomplete-option--${option.code}`"
              :data-option-value="option.code"
              :aria-selected="i === indexCounter"
              :ref="`autocomplete-option-index--${i}`"
              @click="onOptionClick(option)"
            >
              {{ option.name }}
            </li>
          </template>
          <template v-else>
            <li
              id="autocomplete-option--NoResults"
            >
            No results
          </li>
          </template>
        </ul>
        <div aria-live="polite" role="status" class="visually-hidden">
          <span v-if="results.length === 0">No results.</span>
          <span v-else>
            {{ results.length }} results available.
          </span>
        </div>
      </div>
      <button type="button" @click="submit">Submit</button>
    </div>
  </div>
</template>

<script>
/* eslint-disable no-console */
/* eslint-disable max-len */
/* eslint-disable no-plusplus */
/* eslint-disable no-alert */
import countries from '../countries';

export default {
  name: 'HelloWorld',
  data() {
    return {
      keys: {
        enter: 'Enter',
        esc: 'Escape',
        space: 'Space',
        up: 'ArrowUp',
        down: 'ArrowDown',
        tab: 'Tab',
        left: 'ArrowLeft',
        right: 'ArrowRight',
      },
      inputValue: '',
      isFocused: false,
      menuOpen: false,
      results: [],
      indexCounter: -1,
    };
  },
  // computed: {
  //   results() {
  //     return countries.filter(country => country.name.toLowerCase().includes(this.inputValue.toLowerCase()));
  //   },
  // },
  methods: {
    submit() {
      const foundCountry = countries.find(c => c.name === this.inputValue);
      if (foundCountry) {
        alert(`Submitting country ${foundCountry.name}`);
      } else {
        alert('Please submit a country from the autocomplete.');
      }
    },
    onMenuKeyDown(t) {
      switch (t.code) {
        // If the first option is focused, set focus to the text box.
        // Otherwise set focus to the previous option.
        case this.keys.up:
          this.onOptionUpArrow(t);
          break;
        // Focus the next menu option. If it’s the last menu option, do nothing.
        case this.keys.down:
          this.onOptionDownArrow(t);
          break;
        // Select the currently highlighted option and focus the text box.
        case this.keys.enter:
          this.onOptionEnter(t);
          break;
        // Select the currently highlighted option and focus the text box.
        case this.keys.space:
          this.onOptionSpace(t);
          break;
        // Hide the menu and focus the text box.
        case this.keys.esc:
          this.onOptionEscape(t);
          break;
        // Hide the menu.
        case this.keys.tab:
          this.hideMenu();
          this.removeTextBoxFocus();
          break;
        // Focus the text box (so users can continue typing).
        default:
          this.focusTextBox();
      }
    },
    onOptionEnter(e) {
      this.inputValue = this.results[this.indexCounter].name;
      this.hideMenu();
      this.focusTextBox();
      // we don't want form to submit
      e.preventDefault();
    },
    onOptionSpace(e) {
      this.inputValue = this.results[this.indexCounter].name;
      this.hideMenu();
      this.focusTextBox();
      // we don't want a space to be added to text box
      e.preventDefault();
    },
    onOptionEscape() {
      this.clearOptions();
      this.hideMenu();
      this.focusTextBox();
    },
    onOptionDownArrow(e) {
      if (this.indexCounter < this.results.length - 1) {
        this.indexCounter = this.indexCounter + 1;
        this.$nextTick(() => {
          this.$refs[`autocomplete-option-index--${this.indexCounter}`][0].focus();
        });
      }
      e.preventDefault();
    },
    onOptionUpArrow() {
      if (this.indexCounter > 0) {
        this.indexCounter = this.indexCounter - 1;
      } else {
        this.focusTextBox();
        this.hideMenu();
      }
    },
    onTextBoxKeyUp(e) {
      if (e.shiftKey) {
        return;
      }

      switch (e.code) {
        case this.keys.esc:
        case this.keys.up:
        case this.keys.left:
        case this.keys.right:
        case this.keys.space:
        case this.keys.enter:
        case this.keys.tab:
          // ignore otherwise the menu will show
          break;
        case this.keys.down:
          this.onTextBoxDownPressed(e);
          break;
        default:
          this.onTextBoxType(e);
      }
    },
    onTextBoxKeyDown(e) {
      switch (e.code) {
        case this.keys.tab:
          // hide menu
          this.hideMenu();
          this.removeTextBoxFocus();
          break;
        default:
          this.focusTextBox();
      }
    },
    onTextBoxDownPressed(e) {
      let options;
      const value = this.inputValue.trim();
      /*
        When the value is empty show the entire menu
      */
      if (value.length === 0) {
        // get options based on the value
        options = this.getAllOptions();

        // build the menu based on the options
        this.buildMenu(options);

        // show the menu
        this.showMenu();

        this.onOptionDownArrow(e);
        /*
        When there’s a value that doesn’t have
        an exact match show the matching options
      */
      } else {
        // get options based on the value
        options = this.getOptions(value);

        // if there are options
        if (options.length > 0) {
          // build the menu based on the options
          this.buildMenu(options);

          // show the menu
          this.showMenu();

          this.onOptionDownArrow(e);
        }
      }
    },
    onTextBoxType(e) {
      // only show options if user typed something
      const value = e.target.value;

      if (value.trim().length > 0) {
        // get options based on value
        const options = this.getOptions(value.trim().toLowerCase());

        // build the menu based on the options
        this.buildMenu(options);

        // show the menu
        this.showMenu();
      }
    },
    getOptions(inputValue) {
      // return countries.filter(country => country.name.toLowerCase().includes(this.inputValue.toLowerCase()));
      const matches = [];

      // Loop through each of the option elements
      for (let i = 0; i < countries.length; i++) {
        const countryObj = countries[i];
        // if the option has a value and the option’s text node matches the user-typed value
        if (countryObj.code.trim().length > 0 && countryObj.name.toLowerCase().includes(inputValue.toLowerCase())) {
          // push an object representation to the matches array
          matches.push(countryObj);
        }
      }
      return matches;
    },
    getAllOptions() {
      const filtered = [];
      const options = countries;
      let option;
      for (let i = 0; i < options.length; i++) {
        option = options[i];
        const value = option.code;
        if (value.trim().length > 0) {
          filtered.push(option);
        }
      }
      return filtered;
    },
    buildMenu(options) {
      this.clearOptions();
      this.activeOptionId = '';
      this.results = options;
      this.$nextTick(() => {
        document.getElementById('autocomplete-options--destination').scrollTop = 0;
      });
    },
    showMenu() {
      this.menuOpen = true;
    },
    hideMenu() {
      this.menuOpen = false;
      this.activeOptionId = '';
      this.indexCounter = -1;
      this.clearOptions();
    },
    clearOptions() {
      this.results = [];
    },
    removeTextBoxFocus() {
      this.isFocused = false;
    },
    onTextBoxFocus() {
      this.isFocused = true;
    },
    focusTextBox() {
      this.$refs.destination.focus();
    },
    onOptionClick(option) {
      this.inputValue = option.name;
      this.hideMenu();
      this.focusTextBox();
    },
    onDocumentClick(e) {
      // if clicking anywhere but a descendent of the autocomplete, close the menu
      if (!e.target.closest('div.autocomplete')) {
        this.hideMenu();
        this.removeTextBoxFocus();
      } else {
        return e;
      }
      return null;
    },
    onTextBoxClick(event) {
      this.clearOptions();
      const options = this.getAllOptions();
      this.buildMenu(options);
      this.showMenu();
      if (typeof event.currentTarget.select === 'function') {
        event.currentTarget.select();
      }
    },
    onArrowClick() {
      this.clearOptions();
      const options = this.getAllOptions();
      this.buildMenu(options);
      this.showMenu();
      this.focusTextBox();
    },
  },
  mounted() {
    document.addEventListener('click', this.onDocumentClick);
  },
  beforeDestroy() {
    document.removeEventListener('click', this.onDocumentClick);
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1,
h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
/* li {
  display: inline-block;
  margin: 0 10px;
} */
a {
  color: #42b983;
}

.container {
  max-width: 30rem;
  margin: auto;
}

.visually-hidden {
  border: 0 !important;
  clip: rect(0 0 0 0) !important;
  height: 1px !important;
  margin: -1px !important;
  overflow: hidden !important;
  padding: 0 !important;
  position: absolute !important;
  width: 1px !important;
}

.autocomplete {
  position: relative;
}

.autocomplete [type="text"] {
  /* -webkit-appearance: none; */
  border-radius: 0;
  box-sizing: border-box;
  width: 100%;
  font-size: inherit;
  font-family: inherit;
  line-height: 1.25;
  padding: 0.5em;
  border: 2px solid #718096;
}

.autocomplete svg {
  position: absolute;
  right: 0.6em;
  width: 1.5em;
  height: 1.5em;
  top: 0.8em;
  fill: #4a5568;
}

.autocomplete [role="listbox"] {
  margin: 0;
  max-height: 12em;
  overflow-y: auto;
  overflow-y: scroll;
  -webkit-overflow-scrolling: touch;
  position: relative;
  font-size: 100%;
  padding: 0;
  position: absolute;
  top: 2.3em;
  width: 100%;
  background-color: #f7fafc;
  border-radius: 0;
  box-sizing: border-box;
  -moz-box-sizing: border-box;
  -webkit-box-sizing: border-box;
  z-index: 2;
  border: 2px solid #718096;
}

.autocomplete [role="option"] {
  line-height: 22px;
  padding: 0.5em;
  display: block;
  border-bottom: 2px solid #718096;
  outline: 0;
  margin: 0;
}

.hidden {
  display: none;
}

.autocomplete [role="option"][aria-selected="true"] {
  background-color: #005ea5;
  border-color: #005ea5;
  color: #ffffff;
}

.autocomplete-isFocused {
  outline-offset: 0;
  outline: 3px solid #ecc94b;
}
</style>
