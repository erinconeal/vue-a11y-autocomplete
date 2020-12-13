<template>
  <div class="hello">
    <h1>vue-a11y-autocomplete</h1>
    <div class="container">
      <label for="destination">
        <span>Destination</span>
      </label>
      <select
        name="destination"
        aria-hidden="true"
        tabindex="-1"
        class="visually-hidden"
        v-model="selected"
      >
        <option value="">Please select</option>
        <option
          v-for="option in results"
          :key="option.code"
          :value="option.code"
        >
          {{ option.name }}
        </option>
      </select>
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
          :value="inputValue"
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
          <li
            v-for="option in results"
            :key="option.code"
            role="option"
            tabindex="-1"
            aria-selected="true"
            :data-option-value="option.code"
            :id="`autocomplete-option--${option.code}`"
            @click="onOptionClick"
          >
            {{ option.name }}
          </li>
        </ul>
        <div aria-live="polite" role="status" class="visually-hidden">
          {{ results.length }} results available.
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
import countries from "../countries";

export default {
  name: "HelloWorld",
  data() {
    return {
      selected: "",
      keys: {
        enter: 13,
        esc: 27,
        space: 32,
        up: 38,
        down: 40,
        tab: 9,
        left: 37,
        right: 39,
        shift: 16,
      },
      inputValue: "",
      activeOptionId: "",
      isFocused: false,
      menuOpen: false,
    };
  },
  computed: {
    results() {
      return countries.filter(country => country.name.toLowerCase().includes(this.inputValue.toLowerCase()));
    },
  },
  methods: {
    submit() {},
    onMenuKeyDown(t) {
      switch (t.keyCode) {
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
          this.$refs.destination.focus();
      }
    },
    onOptionEnter(t) {
      this.isOptionSelected() && this.selectActiveOption();
      t.preventDefault();
    },
    onOptionSpace() {
      this.isOptionSelected() &&
        (this.selectActiveOption(), t.preventDefault());
    },
    onOptionEscape() {
      this.clearOptions();
      this.hideMenu();
      this.focusTextBox();
    },
    onOptionDownArrow(t) {
      const nextOption = this.getNextOption();
      if (nextOption[0]) {
        this.highlightOption(nextOption);
      }
      t.preventDefault();
    },
    onOptionUpArrow(t) {
      let option;
      // eslint-disable-next-line no-unused-expressions
      this.isOptionSelected() &&
        ((option = this.getPreviousOption()),
        option[0]
          ? this.highlightOption(option)
          : (this.focusTextBox(), this.hideMenu()));
      t.preventDefault();
    },
    isOptionSelected() {
      return this.activeOptionId;
    },
    selectActiveOption() {
      const t = this.getActiveOption();
      this.selectOption(t);
    },
    getActiveOption() {
      return `#${this.activeOptionId}`;
    },
    onTextBoxKeyUp(e) {
      switch (e.keyCode) {
        case this.keys.esc:
        case this.keys.up:
        case this.keys.left:
        case this.keys.right:
        case this.keys.space:
        case this.keys.enter:
        case this.keys.tab:
        case this.keys.shift:
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
      switch (e.keyCode) {
        case this.keys.tab:
          // hide menu
          this.hideMenu();
          this.removeTextBoxFocus();
          break;
        default:
          this.$refs.destination.focus();
      }
    },
    onTextBoxDownPressed(e) {
      console.log("onTextBoxDownPressed", e);
      let option;
      let options;
      const value = this.inputValue.trim();
      /*
        When the value is empty or if it exactly
        matches an option show the entire menu
      */
      if (value.length === 0 || this.isExactMatch(value)) {
        // get options based on the value
        options = this.getAllOptions();

        // build the menu based on the options
        this.buildMenu(options);

        // show the menu
        this.showMenu();

        // retrieve the first option in the menu
        option = this.getFirstOption();

        // highlight the first option
        this.highlightOption(option);

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

          // retrieve the first option in the menu
          option = this.getFirstOption();

          // highlight the first option
          this.highlightOption(option);
        }
      }
    },
    onTextBoxType(e) {
      console.log("onTextBoxType", e);
      // only show options if user typed something
      const value = event.target.value;
      if (value.trim().length > 0) {
        // get options based on value
        const options = this.getOptions(value.trim().toLowerCase());

        // build the menu based on the options
        this.buildMenu(options);

        // show the menu
        this.showMenu();

        // update the live region
        this.updateStatus(options.length);
      }

      // update the select box value which
      // the server uses to process the data
      this.updateSelectBox();
    },
    getOptions(value) {
      const matches = [];

      // Loop through each of the option elements
      // this.select.find('option').each(function(i, el) {
      //   // if the option has a value and the option’s text node matches the user-typed value
      //   if(el.val().trim().length > 0 && el.text().toLowerCase().indexOf(value.toLowerCase()) > -1) {

      //     // push an object representation to the matches array
      //     matches.push({ text: el.text(), value: el.val() });
      //   }
      // });

      return matches;
    },
    getAllOptions() {
      for (
        var t, matches = [], option = this.select.find("option"), i = 0;
        i < option.length;
        i++
      ) {
        (t = option.eq(i)).val().trim().length > 0 &&
          matches.push({
            text: t.text(),
            value: t.val(),
          });
      }
      return matches;
    },
    getFirstOption() {
      return document.getElementById("autocomplete-options--destination")
        .firstChild;
    },
    getPreviousOption() {
      return document.getElementById(this.activeOptionId).previousSibling;
    },
    getNextOption() {
      return document.getElementById(this.activeOptionId).nextSibling;
    },
    highlightOption(option) {
      // if there’s a currently selected option
      if (this.activeOptionId) {
        // get the option
        const activeOption = this.getOptionById(this.activeOptionId);

        // unselect the option
        activeOption.setAttribute("aria-selected", "false");
      }

      // set new option to selected
      option.setAttribute("aria-selected", "true");

      // If the option isn’t visible within the menu
      // if (!this.isElementVisible(option.parent(), option)) {
      //   // make it visible by setting its position inside the menu
      //   option.parent().scrollTop(option.parent().scrollTop() + option.position().top);
      // }

      // store new option for next time
      this.activeOptionId = option[0].id;

      // focus the option
      option.focus();
    },
    isElementVisible() {},
    getOptionById(id) {
      return `#${id}`;
    },
    buildMenu() {},
    showMenu() {
      this.menuOpen = true;
    },
    updateStatus() {},
    updateSelectBox() {
      const t = this.inputValue.trim();
      const matchingSelectOption = this.getMatchingOption(t);
      // TODO: set this.selected value
      // matchingSelectOption ? this.select.val(matchingSelectOption.value) : this.select.val('');
    },
    hideMenu() {
      this.menuOpen = false;
      this.activeOptionId = "";
      this.clearOptions();
    },
    removeTextBoxFocus() {
      this.isFocused = false;
    },
    isExactMatch(option) {
      return this.getMatchingOption(option);
    },
    getMatchingOption(option) {
      let matchedOption = null;
      const options = this.select.find("options");
      for (let i = 0; i < options.length; i++) {
        if (options[i].text.toLowerCase() === option.toLowerCase()) {
          matchedOption = options[i];
          break;
        }
      }
      return matchedOption;
    },
    setValue() {
      // populates the text box and hidden select box
    },
    focusTextBox() {
      this.$refs.destination.focus();
    },
    selectOption(option) {
      const value = option.getAttribute("data-option-value");
      this.setValue(value);
      this.hideMenu();
      this.focusTextBox();
    },
    onOptionClick(e) {
      const option = e.currentTarget;
      this.selectOption(option);
    },
    onDocumentClick(e) {
      console.log("clicking on document", e.target.closest("div.autocomplete"));
      if (!e.target.closest("div.autocomplete")) {
        console.log("clicked outside of autocomplete");
        this.hideMenu();
        this.removeTextBoxFocus();
      }
    },
    onTextBoxClick(t) {
      this.clearOptions();
      const options = this.getAllOptions();
      this.buildMenu(options),
        this.updateStatus(options.length),
        this.showMenu(),
        typeof t.currentTarget.select === "function" &&
          t.currentTarget.select();
    },
    onTextBoxFocus() {
      this.isFocused = true;
    },
    onArrowClick() {},
    clearOptions() {
      console.log("clearOptions called");
    },
  },
  mounted() {
    document.addEventListener("click", this.onDocumentClick);
  },
  beforeDestroy() {
    document.removeEventListener("click", this.onDocumentClick);
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
