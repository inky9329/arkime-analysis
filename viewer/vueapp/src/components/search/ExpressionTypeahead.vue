<template>

  <div class="mb-1"
    v-on-clickaway="onOffFocus">

    <!-- typeahead input -->
    <div class="input-group input-group-sm">
      <span class="input-group-prepend input-group-prepend-fw cursor-help"
        v-b-tooltip.hover.bottomright.d300="'Search Expression'">
        <span class="input-group-text input-group-text-fw">
          <span v-if="!shiftKeyHold"
            class="fa fa-search fa-fw">
          </span>
          <span v-else
            class="query-shortcut">
            Q
          </span>
        </span>
      </span>
      <input
        type="text"
        tabindex="1"
        id="expression"
        ref="expression"
        placeholder="Search"
        v-model="expression"
        v-caret-pos="caretPos"
        v-focus="focusInput"
        @input="debounceExprChange"
        @keydown.enter.prevent.stop="enterClick"
        @keydown.esc.tab.enter.down.up.prevent.stop="keyup($event)"
        class="form-control search-control"
      />
      <span class="input-group-append"
        v-b-tooltip.hover
        title="This is a pretty long search expression, maybe you want to create a shortcut? Click here to go to the shortcut creation page."
        v-if="expression && expression.length > 200">
        <a type="button"
          href="settings#shortcuts"
          class="btn btn-outline-secondary btn-clear-input">
          <span class="fa fa-question-circle">
          </span>
        </a>
      </span>
      <span class="input-group-append">
        <button type="button"
          @click="saveExpression"
          :disabled="!expression"
          v-b-tooltip.hover.bottom
          title="Save this search expression (apply it from the views menu)"
          class="btn btn-outline-secondary btn-clear-input">
          <span class="fa fa-save">
          </span>
        </button>
      </span>
      <span class="input-group-append">
        <button type="button"
          @click="clear"
          :disabled="!expression"
          title="Remove the search text"
          class="btn btn-outline-secondary btn-clear-input">
          <span class="fa fa-close">
          </span>
        </button>
      </span>
    </div> <!-- /typeahead input -->

    <!-- results dropdown -->
    <div id="typeahead-results"
      ref="typeaheadResults"
      class="dropdown-menu typeahead-results"
      v-show="expression && results && results.length">
      <template v-if="autocompletingField">
        <template v-for="(value, key) in fieldHistoryResults">
          <a :id="key+'history'"
            :key="key+'history'"
            class="dropdown-item cursor-pointer"
            :class="{'active':key === activeIdx,'last-history-item':key === fieldHistoryResults.length-1}"
            @click="addToQuery(value)">
            <span class="fa fa-history"></span>&nbsp;
            <strong v-if="value.exp">{{ value.exp }}</strong>
            <strong v-if="!value.exp">{{ value }}</strong>
            <span v-if="value.friendlyName">- {{ value.friendlyName }}</span>
            <span class="fa fa-close pull-right mt-1"
              :title="`Remove ${value.exp} from your field history`"
              @click.stop.prevent="removeFromFieldHistory(value)">
            </span>
          </a>
          <b-tooltip v-if="value.help"
            :key="key+'historytooltip'"
            :target="key+'history'"
            placement="right"
            boundary="window">
            {{ value.help.substring(0, 100) }}
            <span v-if="value.help.length > 100">
              ...
            </span>
          </b-tooltip>
        </template>
      </template>
      <template v-for="(value, key) in results">
        <a :id="key+'item'"
          :key="key+'item'"
          class="dropdown-item cursor-pointer"
          :title="value.help"
          :class="{'active':key+fieldHistoryResults.length === activeIdx}"
          @click="addToQuery(value)">
          <strong v-if="value.exp">{{ value.exp }}</strong>
          <strong v-if="!value.exp">{{ value }}</strong>
          <span v-if="value.friendlyName">- {{ value.friendlyName }}</span>
        </a>
        <b-tooltip v-if="value.help"
          :key="key+'tooltip'"
          :target="key+'item'"
          placement="right"
          boundary="window">
          {{ value.help.substring(0, 100) }}
          <span v-if="value.help.length > 100">
            ...
          </span>
        </b-tooltip>
      </template>
    </div> <!-- /results dropdown -->

    <!-- error -->
    <div class="dropdown-menu typeahead-results"
      v-show="expression && loadingError">
      <a class="dropdown-item text-danger">
        <span class="fa fa-warning"></span>&nbsp;
        Error: {{ loadingError }}
      </a>
    </div> <!-- /error -->

    <!-- loading -->
    <div class="dropdown-menu typeahead-results"
      v-show="expression && loadingValues">
      <a class="dropdown-item">
        <span class="fa fa-spinner fa-spin"></span>&nbsp;
        Loading...
      </a>
    </div> <!-- /loading -->

  </div>

</template>

<script>
import UserService from '../users/UserService';
import FieldService from './FieldService';
import CaretPos from '../utils/CaretPos';
import { mixin as clickaway } from 'vue-clickaway';
import Focus from '../../../../../common/vueapp/Focus';

let tokens;
let timeout;

const operations = ['==', '!=', '<', '<=', '>', '>='];

export default {
  name: 'ExpressionTypeahead',
  mixins: [clickaway],
  directives: { CaretPos, Focus },
  data: function () {
    return {
      activeIdx: -1,
      results: [],
      loadingError: '',
      loadingValues: false,
      caretPos: 0,
      cancellablePromise: null,
      resultsElement: null,
      // field history vars
      fieldHistoryResults: [],
      lastTokenWasField: false,
      autocompletingField: false,
      // saved expression vars
      savedExpressions: []
    };
  },
  computed: {
    expression: {
      get: function () {
        return this.$store.state.expression;
      },
      set: function (newValue) {
        this.$store.commit('setExpression', newValue);
      }
    },
    focusInput: {
      get: function () {
        return this.$store.state.focusSearch;
      },
      set: function (newValue) {
        this.$store.commit('setFocusSearch', newValue);
      }
    },
    shiftKeyHold: function () {
      return this.$store.state.shiftKeyHold;
    },
    views: function () {
      return this.$store.state.views;
    },
    fieldHistory: function () {
      return this.$store.state.fieldhistory;
    },
    fields: function () {
      return this.$store.state.fieldsArr;
    }
  },
  watch: {
    // watch for route update of expression
    '$route.query.expression': function (newVal, oldVal) {
      this.expression = newVal;

      // reset necessary vars
      this.results = null;
      this.fieldHistoryResults = [];
      this.focusInput = true;
      this.activeIdx = -1;

      // notify parent
      this.$emit('changeExpression');
    }
  },
  created: function () {
    if (this.$route.query.expression) {
      this.expression = this.$route.query.expression;
    }
  },
  mounted: function () {
    // set the results element for keyup event handler
    this.resultsElement = this.$refs.typeaheadResults;
  },
  methods: {
    /* exposed page functions ------------------------------------ */
    clear: function () {
      this.expression = undefined;
    },
    saveExpression: function () {
      this.$emit('modView');
    },
    /**
     * Fired when a value from the typeahead menu is selected
     * @param {Object} val The value to be added to the query
     */
    addToQuery: function (val) {
      let str = val;
      if (val.exp) { str = val.exp; }

      const { expression, replacing } = this.rebuildQuery(this.expression, str);
      this.expression = expression;

      if (this.lastTokenWasField) { // add field to history
        this.addFieldToHistory(val);
      }

      this.results = null;
      this.fieldHistoryResults = [];
      this.activeIdx = -1;

      this.$nextTick(() => {
        const expressionInput = document.getElementById('expression');
        if (replacing) {
          const newCaretPos = this.caretPos + 1 + (str.length - replacing.length);
          expressionInput.setSelectionRange(newCaretPos, newCaretPos);
        }
        expressionInput.focus();
      });
    },
    /* Fired when the search input is changed */
    debounceExprChange: function () {
      this.cancelPromise();

      if (timeout) { clearTimeout(timeout); }
      timeout = setTimeout(() => {
        this.changeExpression();
      }, 500);
    },
    /* fired when enter is clicked from the expression typeahead input */
    enterClick: function () {
      // if the activeIdx >= 0, an item in the dropdown is selected
      if (this.activeIdx >= 0) { return; }
      // only apply the expression and clear the results on enter click when
      // not selecting something inside the typeahead dropdown results list
      this.$emit('applyExpression');
      // clear the timeout for the expression input change so it
      // doesn't update the results which opens the results dropdown
      if (timeout) { clearTimeout(timeout); }
      // cancel any queries to get values for fields so that the
      // dropdown isn't populated with more results
      this.cancelPromise();
      // clear out the results to close the dropdown
      this.results = null;
      this.fieldHistoryResults = [];
    },
    /**
     * Watches for keyup events for escape, tab, enter, down, and up keys
     * Pressing the up and down arrows navigate the dropdown.
     * Pressing enter, adds the active item in the dropdown to the query.
     * Pressing escape, removes the typeahead results.
     * @param {Object} e The keydown event fired by the input
     */
    keyup: function (e) {
      let target;

      // always check for escape before anything else
      if (e.keyCode === 27) {
        // if there's a request in progress, cancel it
        this.cancelPromise();

        // if there's no results blur the input
        if (!this.results || !this.results.length) {
          e.target.blur();
          this.focusInput = false;
        }

        this.loadingValues = false;
        this.loadingError = false;
        this.results = null;
        this.fieldHistoryResults = [];
        this.activeIdx = -1;

        return;
      }

      // check for tab click when results are visible
      if (this.results && this.results.length && e.keyCode === 9) {
        // if there is no item in the results is selected, use the first one
        if (this.activeIdx < 0) { this.activeIdx = 0; }

        const result = this.results[this.activeIdx];
        if (result) { this.addToQuery(result); }

        this.cancelPromise();

        this.loadingValues = false;
        this.loadingError = false;
        this.results = null;
        this.fieldHistoryResults = [];
        this.activeIdx = -1;

        return;
      }

      // if there are no results, just check for enter click to remove typeahead
      if (!this.results || this.results.length === 0) {
        if (e.keyCode === 13) {
          this.cancelPromise();

          this.loadingValues = false;
          this.loadingError = false;
          this.results = null;
          this.fieldHistoryResults = [];
          this.activeIdx = -1;
        }

        return;
      }

      if (!this.activeIdx && this.activeIdx !== 0) { this.activeIdx = -1; }

      switch (e.keyCode) {
      case 40: // down arrow
        this.activeIdx = (this.activeIdx + 1) % (this.fieldHistoryResults.length + this.results.length);
        target = this.resultsElement.querySelectorAll('a')[this.activeIdx];
        if (target && target.parentNode) {
          target.parentNode.scrollTop = target.offsetTop;
        }
        break;
      case 38: // up arrow
        this.activeIdx = (this.activeIdx > 0 ? this.activeIdx : (this.fieldHistoryResults.length + this.results.length)) - 1;
        target = this.resultsElement.querySelectorAll('a')[this.activeIdx];
        if (target && target.parentNode) {
          target.parentNode.scrollTop = target.offsetTop;
        }
        break;
      case 13: // enter
        if (this.activeIdx >= 0) {
          let result;
          if (this.activeIdx < this.fieldHistoryResults.length) {
            result = this.fieldHistoryResults[this.activeIdx];
          } else {
            result = this.results[this.activeIdx - this.fieldHistoryResults.length];
          }
          if (result) {
            // need to decrement counter if the user used arrow keys to select
            // result because this increments the counter by 1
            this.caretPos--;
            this.addToQuery(result);
          }
        }
        break;
      }
    },
    /* Removes typeahead results */
    onOffFocus: function () {
      setTimeout(() => {
        if (timeout) { clearTimeout(timeout); }

        this.cancelPromise();

        this.results = null;
        this.fieldHistoryResults = [];
        this.activeIdx = -1;
        this.focusInput = false;
      }, 300);
    },
    /**
     * Removes an item to the field history (and results)
     * @param {object} field The field to remove from the history
     */
    removeFromFieldHistory: function (field) {
      let index = 0;
      for (const historyField of this.fieldHistory) {
        if (historyField.exp === field.exp) {
          break;
        }
        index++;
      }

      // remove the item from the history
      this.fieldHistory.splice(index, 1);

      // find it in the field history results (displayed in the typeahead)
      index = 0;
      for (const historyField of this.fieldHistoryResults) {
        if (historyField.exp === field.exp) {
          break;
        }
        index++;
      }

      // remove the item from the field history results
      this.fieldHistoryResults.splice(index, 1);

      // save the field history for the user
      UserService.saveState({ fields: this.fieldHistory }, 'fieldHistory');
    },
    /* helper functions ------------------------------------------ */
    /**
     * Adds an item to the beginning of the field history
     * @param {object} field The field to add to the history
     */
    addFieldToHistory: function (field) {
      let found = false;

      if (!field) { return found; }

      let index = 0;
      for (const historyField of this.fieldHistory) {
        if (historyField.exp === field.exp) {
          found = true;
          break;
        }
        index++;
      }

      if (found) { // if the field was found, remove it
        this.fieldHistory.splice(index, 1);
      }

      // add the field to the beginning of the list
      this.fieldHistory.unshift(field);

      // if the list is larger than 30 items
      if (this.fieldHistory.length > 30) {
        // remove the last item in the history
        this.fieldHistory.splice(this.fieldHistory.length - 1, 1);
      }

      // save the field history for the user
      UserService.saveState({ fields: this.fieldHistory }, 'fieldHistory');

      return found;
    },
    /**
     * Filters the field history results and sets variables so that the view
     * and other functions know that a field is being searched in the typeahead
     * @param {string} strToMatch The string to compare the field history to
     */
    updateFieldHistoryResults: function (strToMatch) {
      this.lastTokenWasField = true;
      this.autocompletingField = true;
      this.fieldHistoryResults = this.findMatch(strToMatch, this.fieldHistory) || [];
    },
    /* Displays appropriate typeahead suggestions */
    changeExpression: function () {
      this.activeIdx = -1;
      this.results = null;
      this.fieldHistoryResults = [];
      this.loadingValues = false;
      this.loadingError = false;
      this.lastTokenWasField = false;
      this.autocompletingField = false;

      // if the cursor is at a space
      const spaceCP = (this.caretPos > 0 &&
        this.expression[this.caretPos] === ' ');

      let end = this.caretPos;
      const endLen = this.expression.length;
      for (end; end < endLen; ++end) {
        if (this.expression[end] === ' ') {
          break;
        }
      }

      const currentInput = this.expression.substr(0, end);
      tokens = this.splitExpression(currentInput);

      // add the space to the tokens
      if (spaceCP) { tokens.push(' '); }

      let lastToken = tokens[tokens.length - 1];

      // display fields
      if (tokens.length <= 1) {
        this.results = this.findMatch(lastToken, this.fields);
        this.updateFieldHistoryResults(lastToken);
        return;
      }

      // display operators (depending on field type)
      let token = tokens[tokens.length - 2];
      let field = FieldService.getField(token, true);

      if (field) { // add field to the history
        this.addFieldToHistory(field);
      }

      if (field) {
        if (field.type === 'integer') {
          // if at a space, show all operators
          if (tokens[tokens.length - 1] === ' ') {
            this.results = operations;
          } else {
            this.results = this.findMatch(lastToken, operations);
          }
        } else {
          this.results = this.findMatch(lastToken, ['!=', '==']);
        }

        return;
      }

      // save the operator token for possibly adding 'EXISTS!' result
      const operatorToken = token;

      token = tokens[tokens.length - 3];
      field = FieldService.getField(token, true);

      if (!field) {
        if (/^[!<=>]/.test(token)) {
          // if at a space, show all operators
          if (tokens[tokens.length - 1] === ' ') {
            this.results = ['&&', '||'];
          } else {
            this.results = this.findMatch(lastToken, ['&&', '||']);
          }
        } else {
          this.results = this.findMatch(lastToken, this.fields);
          this.updateFieldHistoryResults(lastToken);
        }

        return;
      }

      // Autocomplete view names
      if (field.type === 'viewand') {
        // findMatch expects an object with keys/values
        const views = Object.fromEntries(Object.keys(this.views).map((v) => [v, v]));
        this.results = this.findMatch(lastToken, views);
      }

      // autocomplete variables
      if (/^(\$)/.test(lastToken)) {
        this.loadingValues = true;
        let url = 'api/shortcuts?fieldFormat=true&map=true';
        if (field && field.type) {
          url += `&fieldType=${field.type}`;
        }
        this.$http.get(url).then((response) => {
          this.loadingValues = false;
          const escapedToken = lastToken.replace('$', '\\$');
          this.results = this.findMatch(escapedToken, response.data);
        }).catch((error) => {
          this.loadingValues = false;
          this.loadingError = error.text || error;
        });

        return;
      }

      // Don't try and autocomplete these fields
      if (field.noFacet || field.regex || field.type.match(/textfield/)) { return; }

      let isValueList = false;
      if (/^(\[)/.test(lastToken)) { // it is a list of values
        isValueList = true;
        lastToken = lastToken.substring(1); // remove first char '['
      }

      // regular expressions start with a forward slash
      // lists start with an open square bracket
      // don't issue query for these types of values
      if (/^(\/|\[)/.test(lastToken)) { return; }

      // display values
      // autocomplete country values
      if (/^(country)/.test(token)) {
        this.loadingValues = true;
        FieldService.getCountryCodes().then((result) => {
          this.loadingValues = false;
          this.results = this.findMatch(lastToken, result);
          this.addExistsItem(lastToken, operatorToken);
        }).catch((error) => {
          this.loadingValues = false;
          this.loadingError = error;
        });

        return;
      }

      // if the token is a list of values and there are multiple values entered
      if (isValueList && /(,)/.test(lastToken)) {
        // find the value from , | [ to the cursor position
        let pos = this.caretPos;
        const queryChars = [];
        for (pos; pos >= 0; pos--) {
          const char = this.expression[pos];
          if (char === ',' || char === '[') {
            break;
          }
          queryChars.push(char);
        }
        lastToken = queryChars.reverse().join('');

        if (/(])$/.test(lastToken)) {
          // if theres a closing brace, remove it from the token for querying
          lastToken = lastToken.substring(0, lastToken.length - 1);
        }
      }

      // autocomplete other values after 2 chars
      if (lastToken.trim().length >= 2) {
        const params = { // build parameters for getting value(s)
          autocomplete: true,
          field: field.dbField
        };

        if (this.$route.query.date) {
          params.date = this.$route.query.date;
        } else if (this.$route.query.startTime !== undefined &&
          this.$route.query.stopTime !== undefined) {
          params.startTime = this.$route.query.startTime;
          params.stopTime = this.$route.query.stopTime;
        }

        if (field.type === 'ip') {
          params.expression = token + '==' + lastToken;
        } else {
          params.expression = token + '==' + lastToken + '*';
        }

        this.loadingValues = true;

        this.cancellablePromise = FieldService.getValues(params);

        this.cancellablePromise.promise.then((result) => {
          this.cancellablePromise = null;
          if (result) {
            this.loadingValues = false;
            this.loadingError = '';
            this.results = result;
            this.addExistsItem(lastToken, operatorToken);
          }
        }).catch((error) => {
          this.cancellablePromise = null;
          this.loadingValues = false;
          this.loadingError = error.message || error;
        });
      }
    },
    /**
     * Adds 'EXISTS!' result item to the typeahead options
     * @param {string} strToMatch The string to compare 'EXISTS!' to
     * @param {string} operator   The operator of the expression
     */
    addExistsItem: function (strToMatch, operator) {
      if (operator !== '==' && operator !== '!=') { return; }

      try {
        if ('EXISTS!'.match(new RegExp(strToMatch + '.*'))) {
          this.results.push('EXISTS!');
        }
      } catch (error) {}
    },
    /* aborts a pending promise */
    cancelPromise: function () {
      if (this.cancellablePromise) {
        this.cancellablePromise.source.cancel();
        this.cancellablePromise = null;
        this.loadingValues = false;
        this.loadingError = '';
      }
    },
    /**
     * Finds matching items from an array or map of values
     * @param {string} strToMatch  The string to compare with the values
     * @param {Object} values      Map or Array of values to compare against
     */
    findMatch: function (strToMatch, values) {
      if (!strToMatch || strToMatch === '') { return null; }

      let results = [];
      let exact = false;

      for (const key in values) {
        let str;
        const field = values[key];

        strToMatch = strToMatch.toLowerCase();

        if (field.exp) {
          str = field.exp.toLowerCase();
        } else {
          str = field.toLowerCase();
        }

        if (str === strToMatch) {
          exact = field;
        } else {
          const match = str.match(strToMatch);
          if (match) { results.push(field); }
        }
      }

      // put the exact match at the top (the rest are in the order received)
      if (exact) { results.unshift(exact); }

      if (!results.length) { results = null; }

      return results;
    },
    /**
     * Builds the query by either appending a string onto it or replacing
     * the last token with a string and rebuilding the query string
     * @param {string} q   The query string to be appended to or rebuilt
     * @param {string} str The string to add to the query
     */
    rebuildQuery: function (q, str) {
      let result = '';
      let lastToken = tokens[tokens.length - 1];
      const allTokens = this.splitExpression(q);
      let replacingToken = lastToken;

      if (lastToken === ' ') {
        replacingToken = null; // we're not replacing just adding to the end
        result = q += str + ' ';
      } else { // replace the last token and rebuild query
        let t, i;

        const isArray = /^(\[)/.test(lastToken);
        if (isArray) { // it's an array of values
          let pos = this.caretPos;

          // if were not at the end of the list or we're not right before a comma
          // we need to add a comma after the cursor position so that the list of
          // values can be split into tokens and the appropriate token replaced
          if (this.expression[pos + 1] !== ',' && this.expression[pos + 1] !== ']') {
            const subExpr = tokens.join(' ');
            const tokenCaretPos = this.caretPos - (subExpr.length - lastToken.length);
            if (tokenCaretPos !== lastToken.length - 1) { // add the comma if we're not at the end
              lastToken = lastToken.slice(0, tokenCaretPos + 1) + ',' + lastToken.slice(tokenCaretPos + 1);
            }
          }

          // remove surrounding brackets so the search works
          let hasEndBracket = false;
          if (/(])$/.test(lastToken)) { // remove last char if it's ']'
            hasEndBracket = true;
            lastToken = lastToken.substring(0, lastToken.length - 1);
          }
          const split = lastToken.split(',');
          split[0] = split[0].substring(1); // remove first char (always '[')

          // find the token we're trying to replace by finding the value from
          // , or [ to the cursor position by traversing back from the cursor pos
          const queryChars = [];
          for (pos; pos >= 0; pos--) {
            const char = this.expression[pos];
            if (char === ',' || char === '[') {
              break;
            }
            queryChars.push(char);
          }

          // the token we're looking to replace has been found but is backwards
          replacingToken = queryChars.reverse().join('');

          // find the token to replace in the list of values
          for (i = 0; i < split.length; ++i) {
            if (split[i] === replacingToken) {
              split[i] = str; // replace it with the autocomplete string
              break;
            }
          }

          str = `[${split.join(',')}`; // join back the values with a ,
          if (hasEndBracket) { str += ']'; } // add the ']' back
        }

        tokens[tokens.length - 1] = str;

        for (i = 0; i < tokens.length; ++i) {
          t = tokens[i];
          if (t === ' ') { break; }
          result += t + ' ';
        }

        if (allTokens.length > tokens.length) {
          // add the rest of the tokens (after the autocomplete result was added)
          for (i; i < allTokens.length; ++i) {
            t = allTokens[i];
            if (t === ' ') { break; }
            result += t + ' ';
          }
        }
      }

      return { expression: result, replacing: replacingToken };
    },
    /**
     * Splits a string into tokens
     * @param {string} input The string to be tokenized
     */
    splitExpression: function (input) {
      // replace spaces that are not enclosed in quotes
      input = input.replace(/ (?=([^"]*"[^"]*")*[^"]*$)/g, '');

      const output = [];
      let cur = '';

      for (let i = 0, ilen = input.length; i < ilen; i++) {
        if (/[)(]/.test(input[i])) {
          if (cur !== '') {
            output.push(cur);
          }
          output.push(input[i]);
          cur = '';
        } else if (cur === '') {
          cur += input[i];
        } else if (/[!&|=<>]/.test(cur) || cur === 'EXISTS' || cur === 'EXISTS!') {
          if ((/[&|=<>]/.test(input[i]) && cur !== 'EXISTS!') || (cur === 'EXISTS' && input[i] === '!')) {
            cur += input[i];
          } else {
            output.push(cur);
            cur = input[i];
          }
        } else if (/[!&|=<>]/.test(input[i])) {
          output.push(cur);
          cur = input[i];
        } else {
          cur += input[i];
        }
      }

      if (cur !== '') {
        output.push(cur);
      }

      return output;
    }
  },
  beforeDestroy: function () {
    this.cancelPromise();
    if (timeout) { clearTimeout(timeout); }
  }
};
</script>

<style scoped>
.input-group {
  flex-wrap: none;
  width: auto;
}

.typeahead-results {
  top: initial;
  left: initial;
  display: block;
  overflow-y: auto;
  overflow-x: hidden;
  max-height: 500px;
  margin-left: 35px;
}

.typeahead-results a.last-history-item {
  border-bottom: 1px solid var(--color-gray);
}

@media screen and (max-height: 600px) {
  .typeahead-results {
    max-height: 250px;
  }
}
</style>
