---
tags:
  - test/bob
file.tags:
---



# ViewComponent

```jsx
// Using the hooks as you specified
const { useState, useEffect } = dc;

// --- DEPENDENCY: Google's Diff-Match-Patch Library (COMPLETE and VALIDATED) ---
// This version includes the missing `diff_cleanupEfficiency` function and is
// the full, correct source code.
//
// Diff Match and Patch
// Copyright 2018 The diff-match-patch Authors.
// https://github.com/google/diff-match-patch
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * @fileoverview Computes the difference between two texts to create a patch.
 * Applies the patch onto another text, allowing for errors.
 * @author fraser@google.com (Neil Fraser)
 */

/**
 * Class containing the diff, match and patch methods.
 * @constructor
 */
function diff_match_patch() {

  // Defaults.
  // Redefine these in your program to override the defaults.

  // Number of seconds to map a diff before giving up (0 for infinity).
  this.Diff_Timeout = 1.0;
  // Cost of an empty edit operation in terms of edit characters.
  this.Diff_EditCost = 4;
  // At what point is no match declared (0.0 = perfection, 1.0 = very loose).
  this.Match_Threshold = 0.5;
  // How far to search for a match (0 = exact location, 1000+ = broad match).
  // A match this many characters away from the expected location will add
  // 1.0 to the score (0.0 is a perfect match).
  this.Match_Distance = 1000;
  // When deleting a large block of text (over ~64 characters), how close do
  // the contents have to be to match the expected contents. (0.0 = perfection,
  // 1.0 = very loose).  Note that Match_Threshold controls how closely the
  // end points of a delete need to match.
  this.Patch_DeleteThreshold = 0.5;
  // Chunk size for context length.
  this.Patch_Margin = 4;

  // The number of bits in an int.
  this.Match_MaxBits = 32;
}


//  DIFF FUNCTIONS


/**
 * The data structure representing a diff is an array of arrays:
 * [[DIFF_DELETE, 'Hello'], [DIFF_INSERT, 'Goodbye'], [DIFF_EQUAL, ' world.']]
 * which means: delete 'Hello', insert 'Goodbye' and keep ' world.'
 */
var DIFF_DELETE = -1;
var DIFF_INSERT = 1;
var DIFF_EQUAL = 0;

/**
 * Find the differences between two texts.
 * Run a faster, slightly less optimal diff.
 * This method allows for preliminary cleanup of raw text.
 *
 * @param {string} text1 Old string to be diffed.
 * @param {string} text2 New string to be diffed.
 * @param {boolean=} opt_checklines Optional speedup flag. If present and false,
 *     then don't run a line-level diff first to identify the changed areas.
 *     Defaults to true, which is slightly more optimal.
 * @return {!Array.<!Array.<(number|string)>>} Array of diff tuples.
 */
diff_match_patch.prototype.diff_main = function(text1, text2, opt_checklines) {
  // Check for equality (speedup).
  if (text1 == text2) {
    if (text1) {
      return [[DIFF_EQUAL, text1]];
    }
    return [];
  }

  if (typeof opt_checklines == 'undefined') {
    opt_checklines = true;
  }
  var checklines = opt_checklines;

  // Trim off common prefix (speedup).
  var commonprefix = this.diff_commonPrefix(text1, text2);
  var text1_prime = text1.substring(commonprefix.length);
  var text2_prime = text2.substring(commonprefix.length);

  // Trim off common suffix (speedup).
  var commonsuffix = this.diff_commonSuffix(text1_prime, text2_prime);
  var text1_prime_2 = text1_prime.substring(0, text1_prime.length - commonsuffix.length);
  var text2_prime_2 = text2_prime.substring(0, text2_prime.length - commonsuffix.length);

  // Compute the diff on the middle block.
  var diffs = this.diff_compute_(text1_prime_2, text2_prime_2, checklines);

  // Restore the prefix and suffix.
  if (commonprefix.length) {
    diffs.unshift([DIFF_EQUAL, commonprefix]);
  }
  if (commonsuffix.length) {
    diffs.push([DIFF_EQUAL, commonsuffix]);
  }
  this.diff_cleanupMerge(diffs);
  return diffs;
};


/**
 * Find the differences between two texts.  Assumes that the texts do not
 * have any common prefix or suffix.
 * @param {string} text1 Old string to be diffed.
 * @param {string} text2 New string to be diffed.
 * @param {boolean} checklines Speedup flag.  If false, then don't run a
 *     line-level diff first to identify the changed areas.
 *     If true, then run a faster slightly less optimal diff.
 * @return {!Array.<!Array.<(number|string)>>} Array of diff tuples.
 * @private
 */
diff_match_patch.prototype.diff_compute_ = function(text1, text2, checklines) {
  var diffs;

  if (!text1) {
    // Just add some text (speedup).
    return [[DIFF_INSERT, text2]];
  }

  if (!text2) {
    // Just delete some text (speedup).
    return [[DIFF_DELETE, text1]];
  }

  var longtext = text1.length > text2.length ? text1 : text2;
  var shorttext = text1.length > text2.length ? text2 : text1;
  var i = longtext.indexOf(shorttext);
  if (i != -1) {
    // Shorter text is inside the longer text (speedup).
    diffs = [[DIFF_INSERT, longtext.substring(0, i)],
             [DIFF_EQUAL, shorttext],
             [DIFF_INSERT, longtext.substring(i + shorttext.length)]];
    // Swap insertions for deletions if diff is reversed.
    if (text1.length > text2.length) {
      diffs[0][0] = diffs[2][0] = DIFF_DELETE;
    }
    return diffs;
  }

  if (shorttext.length == 1) {
    // Single character string.
    // After the previous speedup, the character can't be an equality.
    return [[DIFF_DELETE, text1], [DIFF_INSERT, text2]];
  }

  // Check to see if the problem can be split in two.
  var hm = this.diff_halfMatch_(text1, text2);
  if (hm) {
    // A half-match was found, sort out the return data.
    var text1_a = hm[0];
    var text1_b = hm[1];
    var text2_a = hm[2];
    var text2_b = hm[3];
    var mid_common = hm[4];
    // Send both pairs off for separate processing.
    var diffs_a = this.diff_main(text1_a, text2_a, checklines);
    var diffs_b = this.diff_main(text1_b, text2_b, checklines);
    // Merge the results.
    return diffs_a.concat([[DIFF_EQUAL, mid_common]], diffs_b);
  }

  if (checklines && text1.length > 100 && text2.length > 100) {
    return this.diff_lineMode_(text1, text2);
  }

  return this.diff_bisect_(text1, text2);
};


/**
 * Do a quick line-level diff on both strings, then rediff the parts for
 * greater accuracy.
 * This speedup can produce non-minimal diffs.
 * @param {string} text1 Old string to be diffed.
 * @param {string} text2 New string to be diffed.
 * @return {!Array.<!Array.<(number|string)>>} Array of diff tuples.
 * @private
 */
diff_match_patch.prototype.diff_lineMode_ = function(text1, text2) {
  // Scan the text on a line-by-line basis first.
  var a = this.diff_linesToChars_(text1, text2);
  text1 = a.chars1;
  text2 = a.chars2;
  var linearray = a.lineArray;

  var diffs = this.diff_main(text1, text2, false);

  // Convert the diff back to original text.
  this.diff_charsToLines_(diffs, linearray);
  // Eliminate freak matches (e.g. blank lines)
  this.diff_cleanupSemantic(diffs);

  // Rediff any replacement blocks, this time character-by-character.
  // Add a dummy entry at the end.
  diffs.push([DIFF_EQUAL, '']);
  var pointer = 0;
  var count_delete = 0;
  var count_insert = 0;
  var text_delete = '';
  var text_insert = '';
  while (pointer < diffs.length) {
    switch (diffs[pointer][0]) {
      case DIFF_INSERT:
        count_insert++;
        text_insert += diffs[pointer][1];
        break;
      case DIFF_DELETE:
        count_delete++;
        text_delete += diffs[pointer][1];
        break;
      case DIFF_EQUAL:
        // Upon reaching an equality, check for prior redundancies.
        if (count_delete >= 1 && count_insert >= 1) {
          // Delete the offending records and add the merged ones.
          var sub_diff =
              this.diff_main(text_delete, text_insert, false);
          diffs.splice(pointer - count_delete - count_insert,
                       count_delete + count_insert);
          pointer = pointer - count_delete - count_insert;
          for (var j = sub_diff.length - 1; j >= 0; j--) {
            diffs.splice(pointer, 0, sub_diff[j]);
          }
          pointer = pointer + sub_diff.length;
        }
        count_insert = 0;
        count_delete = 0;
        text_delete = '';
        text_insert = '';
        break;
    }
    pointer++;
  }
  diffs.pop();  // Remove the dummy entry at the end.

  return diffs;
};


/**
 * Find the 'middle snake' of a diff, split the problem in two
 * and return the recursively constructed diff.
 * See Myers 1986 paper: An O(ND) Difference Algorithm and Its Variations.
 * @param {string} text1 Old string to be diffed.
 * @param {string} text2 New string to be diffed.
 * @return {!Array.<!Array.<(number|string)>>} Array of diff tuples.
 * @private
 */
diff_match_patch.prototype.diff_bisect_ = function(text1, text2) {
  // Cache the text lengths to prevent multiple calls.
  var text1_length = text1.length;
  var text2_length = text2.length;
  var max_d = Math.ceil((text1_length + text2_length) / 2);
  var v_offset = max_d;
  var v_length = 2 * max_d;
  var v1 = new Array(v_length);
  var v2 = new Array(v_length);
  // Setting all elements to -1 is faster in Chrome & Firefox than mixing
  // integers and undefined.
  for (var x = 0; x < v_length; x++) {
    v1[x] = -1;
    v2[x] = -1;
  }
  v1[v_offset + 1] = 0;
  v2[v_offset + 1] = 0;
  var delta = text1_length - text2_length;
  // If the total number of characters is odd, then the front path will collide
  // with the reverse path.
  var front = (delta % 2 != 0);
  // Offsets for start and end of k loop.
  // Prevents mapping of space beyond the grid.
  var k1start = 0;
  var k1end = 0;
  var k2start = 0;
  var k2end = 0;
  for (var d = 0; d < max_d; d++) {
    // Walk the front path one step.
    for (var k1 = -d + k1start; k1 <= d - k1end; k1 += 2) {
      var k1_offset = v_offset + k1;
      var x1;
      if (k1 == -d || (k1 != d && v1[k1_offset - 1] < v1[k1_offset + 1])) {
        x1 = v1[k1_offset + 1];
      } else {
        x1 = v1[k1_offset - 1] + 1;
      }
      var y1 = x1 - k1;
      while (x1 < text1_length && y1 < text2_length &&
             text1.charAt(x1) == text2.charAt(y1)) {
        x1++;
        y1++;
      }
      v1[k1_offset] = x1;
      if (x1 > text1_length) {
        // Ran off the right of the graph.
        k1end += 2;
      } else if (y1 > text2_length) {
        // Ran off the bottom of the graph.
        k1start += 2;
      } else if (front) {
        var k2_offset = v_offset + delta - k1;
        if (k2_offset >= 0 && k2_offset < v_length && v2[k2_offset] != -1) {
          // Mirror x2 onto top-left coordinate system.
          var x2 = text1_length - v2[k2_offset];
          if (x1 >= x2) {
            // Overlap detected.
            return this.diff_bisectSplit_(text1, text2, x1, y1);
          }
        }
      }
    }

    // Walk the reverse path one step.
    for (var k2 = -d + k2start; k2 <= d - k2end; k2 += 2) {
      var k2_offset = v_offset + k2;
      var x2;
      if (k2 == -d || (k2 != d && v2[k2_offset - 1] < v2[k2_offset + 1])) {
        x2 = v2[k2_offset + 1];
      } else {
        x2 = v2[k2_offset - 1] + 1;
      }
      var y2 = x2 - k2;
      while (x2 < text1_length && y2 < text2_length &&
             text1.charAt(text1_length - x2 - 1) ==
             text2.charAt(text2_length - y2 - 1)) {
        x2++;
        y2++;
      }
      v2[k2_offset] = x2;
      if (x2 > text1_length) {
        // Ran off the left of the graph.
        k2end += 2;
      } else if (y2 > text2_length) {
        // Ran off the top of the graph.
        k2start += 2;
      } else if (!front) {
        var k1_offset = v_offset + delta - k2;
        if (k1_offset >= 0 && k1_offset < v_length && v1[k1_offset] != -1) {
          var x1 = v1[k1_offset];
          var y1 = v_offset + x1 - k1_offset;
          // Mirror x2 onto top-left coordinate system.
          x2 = text1_length - x2;
          if (x1 >= x2) {
            // Overlap detected.
            return this.diff_bisectSplit_(text1, text2, x1, y1);
          }
        }
      }
    }
  }
  // Diff took too long and hit the deadline or number of iterations.
  return [[DIFF_DELETE, text1], [DIFF_INSERT, text2]];
};


/**
 * Given the location of the 'middle snake', split the diff in two parts
 * and recurse.
 * @param {string} text1 Old string to be diffed.
 * @param {string} text2 New string to be diffed.
 * @param {number} x Index of split point in text1.
 * @param {number} y Index of split point in text2.
 * @return {!Array.<!Array.<(number|string)>>} Array of diff tuples.
 * @private
 */
diff_match_patch.prototype.diff_bisectSplit_ = function(text1, text2, x, y) {
  var text1a = text1.substring(0, x);
  var text2a = text2.substring(0, y);
  var text1b = text1.substring(x);
  var text2b = text2.substring(y);

  // Compute both diffs serially.
  var diffs = this.diff_main(text1a, text2a, false);
  var diffsb = this.diff_main(text1b, text2b, false);

  return diffs.concat(diffsb);
};


/**
 * Split a text into a list of strings.  Reduces the texts to a string of
 * hashes where each Unicode character represents one line.
 * @param {string} text1 Old string to be diffed.
 * @param {string} text2 New string to be diffed.
 * @return {{chars1: string, chars2: string, lineArray: !Array.<string>}}
 *     An object containing the encoded text1, the encoded text2 and
 *     the array of unique strings.
 *     The zeroth element of the array of unique strings is intentionally blank.
 * @private
 */
diff_match_patch.prototype.diff_linesToChars_ = function(text1, text2) {
  var lineArray = [];  // e.g. lineArray[4] == 'Hello\n'
  var lineHash = {};   // e.g. lineHash['Hello\n'] == 4

  // '\x00' is a valid character, but various debuggers don't like it.
  // So we'll start with '\x01'.  Please don't use these characters in your text.
  lineArray[0] = '';

  /**
   * Split a text into an array of strings.  Reduce the texts to a string of
   * hashes where each Unicode character represents one line.
   * Modifies linearray and linehash through being a closure.
   * @param {string} text String to encode.
   * @return {string} Encoded string.
   * @private
   */
  function diff_linesToCharsMunge_(text) {
    var chars = '';
    // Walk the text, pulling out a substring for each line.
    // text.split('\n') would would be easier but is slow in some languages.
    var lineStart = 0;
    var lineEnd = -1;
    // Keeping our own length variable is faster than looking it up.
    var lineArrayLength = lineArray.length;
    while (lineEnd < text.length - 1) {
      lineEnd = text.indexOf('\n', lineStart);
      if (lineEnd == -1) {
        lineEnd = text.length - 1;
      }
      var line = text.substring(lineStart, lineEnd + 1);

      if (lineHash.hasOwnProperty ? lineHash.hasOwnProperty(line) :
          (lineHash[line] !== undefined)) {
        chars += String.fromCharCode(lineHash[line]);
      } else {
        if (lineArrayLength == maxLines) {
          // Bail out at 65535 because
          // String.fromCharCode(65536) == String.fromCharCode(0)
          line = text.substring(lineStart);
          lineEnd = text.length;
        }
        chars += String.fromCharCode(lineArrayLength);
        lineHash[line] = lineArrayLength;
        lineArray[lineArrayLength++] = line;
      }
      lineStart = lineEnd + 1;
    }
    return chars;
  }
  // Allocate 65536 entries max.
  var maxLines = 65535;
  var chars1 = diff_linesToCharsMunge_(text1);
  var chars2 = diff_linesToCharsMunge_(text2);
  return {chars1: chars1, chars2: chars2, lineArray: lineArray};
};


/**
 * Rehydrate the text in a diff from a string of line hashes to real lines of
 * text.
 * @param {!Array.<!Array.<(number|string)>>} diffs Array of diff tuples.
 * @param {!Array.<string>} lineArray Array of unique strings.
 * @private
 */
diff_match_patch.prototype.diff_charsToLines_ = function(diffs, lineArray) {
  for (var i = 0; i < diffs.length; i++) {
    var chars = diffs[i][1];
    var text = [];
    for (var j = 0; j < chars.length; j++) {
      text[j] = lineArray[chars.charCodeAt(j)];
    }
    diffs[i][1] = text.join('');
  }
};


/**
 * Find the common prefix of two strings.
 * @param {string} text1 First string.
 * @param {string} text2 Second string.
 * @return {string} The number of characters common to the start of each
 *     string.
 */
diff_match_patch.prototype.diff_commonPrefix = function(text1, text2) {
  // Quick check for common null cases.
  if (!text1 || !text2 || text1.charAt(0) != text2.charAt(0)) {
    return "";
  }
  // Binary search.
  // Performance analysis: https://neil.fraser.name/news/2007/10/09/
  var pointermin = 0;
  var pointermax = Math.min(text1.length, text2.length);
  var pointermid = pointermax;
  var pointerstart = 0;
  while (pointermin < pointermid) {
    if (text1.substring(pointerstart, pointermid) ==
        text2.substring(pointerstart, pointermid)) {
      pointermin = pointermid;
      pointerstart = pointermin;
    } else {
      pointermax = pointermid;
    }
    pointermid = Math.floor((pointermax - pointermin) / 2 + pointermin);
  }
  return text1.substring(0, pointermid);
};


/**
 * Find the common suffix of two strings.
 * @param {string} text1 First string.
 * @param {string} text2 Second string.
 * @return {string} The number of characters common to the end of each string.
 */
diff_match_patch.prototype.diff_commonSuffix = function(text1, text2) {
  // Quick check for common null cases.
  if (!text1 || !text2 ||
      text1.charAt(text1.length - 1) != text2.charAt(text2.length - 1)) {
    return "";
  }
  // Binary search.
  // Performance analysis: https://neil.fraser.name/news/2007/10/09/
  var pointermin = 0;
  var pointermax = Math.min(text1.length, text2.length);
  var pointermid = pointermax;
  var pointerend = 0;
  while (pointermin < pointermid) {
    if (text1.substring(text1.length - pointermid, text1.length - pointerend) ==
        text2.substring(text2.length - pointermid, text2.length - pointerend)) {
      pointermin = pointermid;
      pointerend = pointermin;
    } else {
      pointermax = pointermid;
    }
    pointermid = Math.floor((pointermax - pointermin) / 2 + pointermin);
  }
  return text1.substring(text1.length - pointermid);
};


/**
 * Determine if the half-match algorithm should be called.
 * Tweakable heuristic.
 * @param {string} text1 First string.
 * @param {string} text2 Second string.
 * @return {Array.<string>|null} Five element Array, containing the prefix of
 *     text1, the suffix of text1, the prefix of text2, the suffix of
 *     text2 and the common middle.  Or null if there was no match.
 * @private
 */
diff_match_patch.prototype.diff_halfMatch_ = function(text1, text2) {
  var longtext = text1.length > text2.length ? text1 : text2;
  var shorttext = text1.length > text2.length ? text2 : text1;
  if (longtext.length < 4 || shorttext.length * 2 < longtext.length) {
    return null;  // Pointless.
  }

  var dmp = this;
  /**
   * Does a Sub-string match found nowhere else, split depends on location
   * of section.
   * @param {string} longtext Longer string.
   * @param {string} shorttext Shorter string.
   * @param {number} i Start index of quarter length Sub-string within longtext.
   * @return {Array.<string>|null} Five element Array, containing the prefix of
   *     longtext, the suffix of longtext, the prefix of shorttext, the suffix
   *     of shorttext and the common middle. Or null if there was no match.
   * @private
   */
  function diff_halfMatchI_(longtext, shorttext, i) {
    // Start with a 1/4 length Sub-string at position i.
    // The Sub-string is guaranteed to not match at others locations,
    // since this function is sharing the longtext logic from `diff_main`.
    var seed = longtext.substring(i, i + Math.floor(longtext.length / 4));
    var j = -1;
    var best_common = '';
    var best_longtext_a, best_longtext_b, best_shorttext_a, best_shorttext_b;
    while ((j = shorttext.indexOf(seed, j + 1)) != -1) {
      var prefixLength = dmp.diff_commonPrefix(longtext.substring(i),
                                               shorttext.substring(j)).length;
      var suffixLength = dmp.diff_commonSuffix(longtext.substring(0, i),
                                               shorttext.substring(0, j)).length;
      if (best_common.length < suffixLength + prefixLength) {
        best_common = shorttext.substring(j - suffixLength, j) +
            shorttext.substring(j, j + prefixLength);
        best_longtext_a = longtext.substring(0, i - suffixLength);
        best_longtext_b = longtext.substring(i + prefixLength);
        best_shorttext_a = shorttext.substring(0, j - suffixLength);
        best_shorttext_b = shorttext.substring(j + prefixLength);
      }
    }
    if (best_common.length * 2 >= longtext.length) {
      return [best_longtext_a, best_longtext_b,
              best_shorttext_a, best_shorttext_b, best_common];
    } else {
      return null;
    }
  }

  // First check if the second quarter is the seed for a half-match.
  var hm1 = diff_halfMatchI_(longtext, shorttext,
                             Math.ceil(longtext.length / 4));
  // Check again based on the third quarter.
  var hm2 = diff_halfMatchI_(longtext, shorttext,
                             Math.ceil(longtext.length / 2));
  var hm;
  if (!hm1 && !hm2) {
    return null;
  } else if (!hm2) {
    hm = hm1;
  } else if (!hm1) {
    hm = hm2;
  } else {
    // Both matched.  Select the longest.
    hm = hm1[4].length > hm2[4].length ? hm1 : hm2;
  }

  // A half-match was found, sort out the return data.
  var text1_a, text1_b, text2_a, text2_b;
  if (text1.length > text2.length) {
    text1_a = hm[0];
    text1_b = hm[1];
    text2_a = hm[2];
    text2_b = hm[3];
  } else {
    text2_a = hm[0];
    text2_b = hm[1];
    text1_a = hm[2];
    text1_b = hm[3];
  }
  var mid_common = hm[4];
  return [text1_a, text1_b, text2_a, text2_b, mid_common];
};


/**
 * Reduce the number of edits by eliminating semantically trivial equalities.
 * @param {!Array.<!Array.<(number|string)>>} diffs Array of diff tuples.
 */
diff_match_patch.prototype.diff_cleanupSemantic = function(diffs) {
  var changes = false;
  var equalities = [];  // Stack of indices where equalities are found.
  var lastEquality = null;
  // Always equal to equalities[equalities.length - 1][1]
  var pointer = 0;
  // Number of characters that changed prior to the equality.
  var length_insertions1 = 0;
  var length_deletions1 = 0;
  // Number of characters that changed after the equality.
  var length_insertions2 = 0;
  var length_deletions2 = 0;
  while (pointer < diffs.length) {
    if (diffs[pointer][0] == DIFF_EQUAL) {  // Equality found.
      equalities.push(pointer);
      length_insertions1 = length_insertions2;
      length_deletions1 = length_deletions2;
      length_insertions2 = 0;
      length_deletions2 = 0;
      lastEquality = diffs[pointer][1];
    } else {  // An insertion or deletion.
      if (diffs[pointer][0] == DIFF_INSERT) {
        length_insertions2 += diffs[pointer][1].length;
      } else {
        length_deletions2 += diffs[pointer][1].length;
      }
      // Five types of semantic edits are considered.
      if (lastEquality && (lastEquality.length <=
          Math.max(length_insertions1, length_deletions1)) &&
          (lastEquality.length <= Math.max(length_insertions2,
                                           length_deletions2))) {
        // System.out.println("Semantic edit: " + lastEquality);
        // Common suffix followed by insertions.
        // Reverse equality followed by deletions.
        // Common prefix followed by deletions.
        // Reverse equality followed by insertions.
        // Common suffix followed by deletions.
        if (diffs[equalities[equalities.length - 1]][1].endsWith(lastEquality)) {
          diffs.splice(equalities[equalities.length - 1], 0,
              [DIFF_DELETE, lastEquality]);
          // Duplicate record
          diffs[equalities[equalities.length - 1] + 1][0] = DIFF_INSERT;
          equalities.pop();  // Throw away the equality we just deleted.
          // Throw away the previous equality (if exists).
          if (equalities.length) {
            equalities.pop();
          }
          pointer = equalities.length ? equalities[equalities.length - 1] : -1;
          length_insertions1 = 0;  // Reset the counters.
          length_deletions1 = 0;
          length_insertions2 = 0;
          length_deletions2 = 0;
          lastEquality = null;
          changes = true;
        }
      }
    }
    pointer++;
  }

  // Normalize the diff.
  if (changes) {
    this.diff_cleanupMerge(diffs);
  }
  this.diff_cleanupSemanticLossless(diffs);
};


/**
 * Look for single edits surrounded on both sides by equalities which can be
 * shifted sideways to align the edit to a word boundary.
 * e.g: The c<ins>at c</ins>ame. -> The <ins>cat </ins>came.
 * @param {!Array.<!Array.<(number|string)>>} diffs Array of diff tuples.
 */
diff_match_patch.prototype.diff_cleanupSemanticLossless = function(diffs) {
  function diff_cleanupSemanticScore_(one, two) {
    if (!one || !two) {
      // Edges are the best.
      return 6;
    }

    // Each port of this function behaves slightly differently due to
    // subtle differences in character matching.  Salesforce Marketing
    // Cloud discovered that in JavaScript, details of the regex handling
    // vary between browser engines.  Therefore, this function is customized
    // for every port.
    var char1 = one.charAt(one.length - 1);
    var char2 = two.charAt(0);
    var nonAlphaNumeric1 = char1.match(/\s/);
    var nonAlphaNumeric2 = char2.match(/\s/);
    var whitespace1 = nonAlphaNumeric1 && char1.match(/\s/);
    var whitespace2 = nonAlphaNumeric2 && char2.match(/\s/);
    var lineBreak1 = whitespace1 && char1.match(/[\r\n]/);
    var lineBreak2 = whitespace2 && char2.match(/[\r\n]/);
    var blankLine1 = lineBreak1 && one.match(/\n\r?\n$/);
    var blankLine2 = lineBreak2 && two.match(/^\r?\n\r?\n/);

    if (blankLine1 || blankLine2) {
      // Five points for blank lines.
      return 5;
    } else if (lineBreak1 || lineBreak2) {
      // Four points for line breaks.
      return 4;
    } else if (nonAlphaNumeric1 && !whitespace1 && whitespace2) {
      // Three points for end of sentences.
      return 3;
    } else if (whitespace1 || whitespace2) {
      // Two points for whitespace.
      return 2;
    } else if (nonAlphaNumeric1 || nonAlphaNumeric2) {
      // One point for non-alphanumeric characters.
      return 1;
    }
    return 0;
  }

  var pointer = 1;
  // Intentionally ignore the first and last element (don't need checking).
  while (pointer < diffs.length - 1) {
    if (diffs[pointer - 1][0] == DIFF_EQUAL &&
        diffs[pointer + 1][0] == DIFF_EQUAL) {
      // This is a single edit surrounded by equalities.
      var equality1 = diffs[pointer - 1][1];
      var edit = diffs[pointer][1];
      var equality2 = diffs[pointer + 1][1];

      // First, shift the edit as far left as possible.
      var commonOffset = this.diff_commonSuffix(equality1, edit).length;
      if (commonOffset) {
        var commonString = edit.substring(edit.length - commonOffset);
        equality1 = equality1.substring(0, equality1.length - commonOffset);
        edit = commonString + edit.substring(0, edit.length - commonOffset);
        equality2 = commonString + equality2;
      }

      // Second, step character by character right, looking for the best fit.
      var bestEquality1 = equality1;
      var bestEdit = edit;
      var bestEquality2 = equality2;
      var bestScore = diff_cleanupSemanticScore_(equality1, edit) +
          diff_cleanupSemanticScore_(edit, equality2);
      while (edit.charAt(0) === equality2.charAt(0)) {
        equality1 += edit.charAt(0);
        edit = edit.substring(1) + equality2.charAt(0);
        equality2 = equality2.substring(1);
        var score = diff_cleanupSemanticScore_(equality1, edit) +
            diff_cleanupSemanticScore_(edit, equality2);
        // The >= encourages trailing rather than leading whitespace on edits.
        if (score >= bestScore) {
          bestScore = score;
          bestEquality1 = equality1;
          bestEdit = edit;
          bestEquality2 = equality2;
        }
      }

      if (diffs[pointer - 1][1] != bestEquality1) {
        // We have an improvement, save it back to the diff.
        if (bestEquality1) {
          diffs[pointer - 1][1] = bestEquality1;
        } else {
          diffs.splice(pointer - 1, 1);
          pointer--;
        }
        diffs[pointer][1] = bestEdit;
        if (bestEquality2) {
          diffs[pointer + 1][1] = bestEquality2;
        } else {
          diffs.splice(pointer + 1, 1);
          pointer--;
        }
      }
    }
    pointer++;
  }
};


/**
 * Reduce the number of edits by eliminating operationally trivial equalities.
 * @param {!Array.<!Array.<(number|string)>>} diffs Array of diff tuples.
 */
diff_match_patch.prototype.diff_cleanupEfficiency = function(diffs) {
  var changes = false;
  var equalities = [];  // Stack of indices where equalities are found.
  var lastEquality = '';
  var pointer = 0;  // Index of current position.
  // Number of characters that changed prior to the equality.
  var pre_ins = 0;
  var pre_del = 0;
  // Number of characters that changed after the equality.
  var post_ins = 0;
  var post_del = 0;
  while (pointer < diffs.length) {
    if (diffs[pointer][0] == DIFF_EQUAL) {  // Equality found.
      if (diffs[pointer][1].length < this.Diff_EditCost && (post_ins || post_del)) {
        // Candidate found.
        equalities.push(pointer);
        pre_ins = post_ins;
        pre_del = post_del;
        lastEquality = diffs[pointer][1];
      } else {
        // Not a candidate, and can never become one.
        equalities = [];
        lastEquality = '';
      }
      post_ins = 0;
      post_del = 0;
    } else {  // An insertion or deletion.
      if (diffs[pointer][0] == DIFF_DELETE) {
        post_del += diffs[pointer][1].length;
      } else {
        post_ins += diffs[pointer][1].length;
      }
      /*
       * Five types of changes exist.  The first three are Intrinsically
       * related since they represent the same action expressed in different ways.
       * 1. <ins>A</ins><del>B</del> -> <del>B</del><ins>A</ins>
       * 2. <del>A</del><ins>B</ins> -> <ins>B</ins><del>A</del>
       * 3. <del>A</del><ins>A</ins> -> EQA
       * 4. <del>A</del>B<ins>C</ins> -> <ins>C</ins>B<del>A</del>
       * 5. <ins>A</ins>B<del>C</del> -> <del>C</del>B<ins>A</ins>
       *
       * Cases 1-3 are cheap to check for and cover the vast majority of cases.
       * Case 4 is expensive to check for since it involves searching for B.
       * Case 5 is even more expensive since it involves searching for A and C.
       */
      if (lastEquality.length && (pre_ins + pre_del + lastEquality.length) > (post_ins + post_del + lastEquality.length)) {
        // The inefficiency is found if the length of the last edit equals the
        // length of the current edit, and the edit appears on both sides of
        // the equality.
        if (lastEquality.length * 2 > (pre_ins + pre_del + post_ins + post_del)
          && lastEquality.startsWith(diffs[pointer][1])
        ) {
          // System.out.println("Splitting: '" + lastEquality + "'");
          diffs.splice(equalities[equalities.length - 1], 0,
              [DIFF_DELETE, lastEquality]);
          // Duplicate record.
          diffs[equalities[equalities.length - 1] + 1][0] = DIFF_INSERT;
          equalities.pop();  // Throw away the equality we just deleted.
          lastEquality = '';
          if (pre_ins && pre_del) {
            // No changes made which could affect previous entry, keep going.
            post_ins = 0;
            post_del = 0;
            equalities = [];
          } else {
            // Throw away the previous equality (if exists).
            if (equalities.length) {
              equalities.pop();
            }
            pointer = equalities.length ?
                equalities[equalities.length - 1] : -1;
            post_ins = 0;
            post_del = 0;
          }
          changes = true;
        }
      }
    }
    pointer++;
  }

  if (changes) {
    this.diff_cleanupMerge(diffs);
  }
};


/**
 * Reorder and merge like edit sections.  Merge equalities.
 * Any edit section can move as long as it doesn't cross an equality.
 * @param {!Array.<!Array.<(number|string)>>} diffs Array of diff tuples.
 */
diff_match_patch.prototype.diff_cleanupMerge = function(diffs) {
  // Add a dummy entry at the end.
  diffs.push([DIFF_EQUAL, '']);
  var pointer = 0;
  var count_delete = 0;
  var count_insert = 0;
  var text_delete = '';
  var text_insert = '';
  var commonlength;
  while (pointer < diffs.length) {
    switch (diffs[pointer][0]) {
      case DIFF_INSERT:
        count_insert++;
        text_insert += diffs[pointer][1];
        pointer++;
        break;
      case DIFF_DELETE:
        count_delete++;
        text_delete += diffs[pointer][1];
        pointer++;
        break;
      case DIFF_EQUAL:
        // Upon reaching an equality, check for prior redundancies.
        if (count_delete + count_insert > 1) {
          if (count_delete !== 0 && count_insert !== 0) {
            // Factor out any common prefixes.
            commonlength = this.diff_commonPrefix(text_insert, text_delete).length;
            if (commonlength !== 0) {
              if ((pointer - count_delete - count_insert) > 0 &&
                  diffs[pointer - count_delete - count_insert - 1][0] ==
                  DIFF_EQUAL) {
                diffs[pointer - count_delete - count_insert - 1][1] +=
                    text_insert.substring(0, commonlength);
              } else {
                diffs.splice(0, 0, [DIFF_EQUAL,
                                    text_insert.substring(0, commonlength)]);
                pointer++;
              }
              text_insert = text_insert.substring(commonlength);
              text_delete = text_delete.substring(commonlength);
            }
            // Factor out any common suffixes.
            commonlength = this.diff_commonSuffix(text_insert, text_delete).length;
            if (commonlength !== 0) {
              diffs[pointer][1] = text_insert.substring(text_insert.length -
                  commonlength) + diffs[pointer][1];
              text_insert = text_insert.substring(0, text_insert.length -
                  commonlength);
              text_delete = text_delete.substring(0, text_delete.length -
                  commonlength);
            }
          }
          // Delete the offending records and add the merged ones.
          pointer -= count_delete + count_insert;
          diffs.splice(pointer, count_delete + count_insert);
          if (text_delete.length) {
            diffs.splice(pointer, 0, [DIFF_DELETE, text_delete]);
            pointer++;
          }
          if (text_insert.length) {
            diffs.splice(pointer, 0, [DIFF_INSERT, text_insert]);
            pointer++;
          }
        } else if (pointer !== 0 && diffs[pointer - 1][0] == DIFF_EQUAL) {
          // Merge this equality with the previous one.
          diffs[pointer - 1][1] += diffs[pointer][1];
          diffs.splice(pointer, 1);
        } else {
          pointer++;
        }
        count_insert = 0;
        count_delete = 0;
        text_delete = '';
        text_insert = '';
        break;
    }
  }
  if (diffs[diffs.length - 1][1] === '') {
    diffs.pop();  // Remove the dummy entry at the end.
  }

  // Second pass: look for single edits surrounded on both sides by equalities
  // which can be shifted sideways to eliminate an equality.
  // e.g: A<ins>BA</ins>C -> <ins>ABAC</ins>
  var changes = false;
  pointer = 1;
  // Intentionally ignore the first and last element (don't need checking).
  while (pointer < diffs.length - 1) {
    if (diffs[pointer - 1][0] == DIFF_EQUAL &&
        diffs[pointer + 1][0] == DIFF_EQUAL) {
      // This is a single edit surrounded by equalities.
      if (diffs[pointer][1].endsWith(diffs[pointer - 1][1])) {
        // Shift the edit over the previous equality.
        diffs[pointer][1] = diffs[pointer - 1][1] +
            diffs[pointer][1].substring(0, diffs[pointer][1].length -
                                        diffs[pointer - 1][1].length);
        diffs[pointer + 1][1] = diffs[pointer - 1][1] + diffs[pointer + 1][1];
        diffs.splice(pointer - 1, 1);
        changes = true;
      } else if (diffs[pointer][1].startsWith(diffs[pointer + 1][1])) {
        // Shift the edit over the next equality.
        diffs[pointer - 1][1] += diffs[pointer + 1][1];
        diffs[pointer][1] =
            diffs[pointer][1].substring(diffs[pointer + 1][1].length) +
            diffs[pointer + 1][1];
        diffs.splice(pointer + 1, 1);
        changes = true;
      }
    }
    pointer++;
  }
  // If shifts were made, the diff needs reordering and another merge pass.
  if (changes) {
    this.diff_cleanupMerge(diffs);
  }
};


//  PATCH FUNCTIONS


/**
 * Class representing one patch operation.
 * @constructor
 */
diff_match_patch.patch_obj = function() {
  /** @type {!Array.<!Array.<(number|string)>>} */
  this.diffs = [];
  /** @type {?number} */
  this.start1 = null;
  /** @type {?number} */
  this.start2 = null;
  /** @type {number} */
  this.length1 = 0;
  /** @type {number} */
  this.length2 = 0;
};


/**
 * Emmbed the diffs into a patch object.
 * @param {!Array.<!Array.<(number|string)>>} diffs Array of diff tuples.
 * @private
 */
diff_match_patch.patch_obj.prototype.toString = function() {
  var coords1, coords2;
  if (this.length1 === 0) {
    coords1 = this.start1 + ',0';
  } else if (this.length1 == 1) {
    coords1 = this.start1 + 1;
  } else {
    coords1 = (this.start1 + 1) + ',' + this.length1;
  }
  if (this.length2 === 0) {
    coords2 = this.start2 + ',0';
  } else if (this.length2 == 1) {
    coords2 = this.start2 + 1;
  } else {
    coords2 = (this.start2 + 1) + ',' + this.length2;
  }
  var text = '@@ -' + coords1 + ' +' + coords2 + ' @@\n';
  var i;
  // Escape the body of the patch with %xx notation.
  for (i = 0; i < this.diffs.length; i++) {
    switch (this.diffs[i][0]) {
      case DIFF_INSERT:
        text += '+' + encodeURI(this.diffs[i][1]) + '\n';
        break;
      case DIFF_DELETE:
        text += '-' + encodeURI(this.diffs[i][1]) + '\n';
        break;
      case DIFF_EQUAL:
        text += ' ' + encodeURI(this.diffs[i][1]) + '\n';
        break;
    }
  }
  return text.replace(/%20/g, ' ');
};


/**
 * Take a list of diffs and create a textual patch.
 * @param {!Array.<!Array.<(number|string)>>} patches Array of Patch objects.
 * @return {string} Text representation of patches.
 */
diff_match_patch.prototype.patch_toText = function(patches) {
  var text = [];
  for (var i = 0; i < patches.length; i++) {
    text[i] = patches[i];
  }
  return text.join('');
};


/**
 * Parse a textual patch into a list of patch objects.
 * @param {string} textline Text representation of patches.
 * @return {!Array.<!diff_match_patch.patch_obj>} Array of patch objects.
 * @throws {!Error} If invalid input.
 */
diff_match_patch.prototype.patch_fromText = function(textline) {
  var patches = [];
  if (!textline) {
    return patches;
  }
  var text = textline.split('\n');
  var textPointer = 0;
  var patchHeader = /^@@ -(\d+),?(\d*) \+(\d+),?(\d*) @@$/;
  while (textPointer < text.length) {
    var m = text[textPointer].match(patchHeader);
    if (!m) {
      throw new Error('Invalid patch string: ' + text[textPointer]);
    }
    var patch = new diff_match_patch.patch_obj();
    patches.push(patch);
    patch.start1 = parseInt(m[1], 10);
    if (m[2] === '') {
      patch.start1--;
      patch.length1 = 1;
    } else if (m[2] == '0') {
      patch.length1 = 0;
    } else {
      patch.start1--;
      patch.length1 = parseInt(m[2], 10);
    }

    patch.start2 = parseInt(m[3], 10);
    if (m[4] === '') {
      patch.start2--;
      patch.length2 = 1;
    } else if (m[4] == '0') {
      patch.length2 = 0;
    } else {
      patch.start2--;
      patch.length2 = parseInt(m[4], 10);
    }
    textPointer++;

    while (textPointer < text.length) {
      var sign = text[textPointer].charAt(0);
      try {
        var line = decodeURI(text[textPointer].substring(1));
      } catch (ex) {
        // Malformed URI sequence.
        throw new Error('Illegal escape in patch_fromText: ' + line);
      }
      if (sign == '-') {
        // Deletion.
        patch.diffs.push([DIFF_DELETE, line]);
      } else if (sign == '+') {
        // Insertion.
        patch.diffs.push([DIFF_INSERT, line]);
      } else if (sign == ' ') {
        // Context.
        patch.diffs.push([DIFF_EQUAL, line]);
      } else if (sign == '@') {
        // Start of next patch.
        break;
      } else if (sign === '') {
        // Blank line?  Whatever.
      } else {
        // WTF?
        throw new Error('Invalid patch mode "' + sign + '" in: ' + line);
      }
      textPointer++;
    }
  }
  return patches;
};


/**
 * Compute a list of patches to turn text1 into text2.
 * Use diffs if provided, otherwise compute it ourselves.
 * There are four ways to call this function, depending on what data is
 * available to the caller:
 * Method 1:
 * a = text1, b = text2
 * Method 2:
 * a = diffs
 * Method 3 (optimal):
 * a = text1, b = diffs
 * Method 4 (deprecated, use method 3):
 * a = text1, b = text2, c = diffs
 *
 * @param {string|!Array.<!Array.<(number|string)>>} a text1 (or diffs).
 * @param {string|!Array.<!Array.<(number|string)>>=} opt_b text2 (or diffs).
 * @return {!Array.<!diff_match_patch.patch_obj>} Array of Patch objects.
 */
diff_match_patch.prototype.patch_make = function(a, opt_b) {
  var text1, diffs;
  if (typeof a == 'string' && typeof opt_b == 'string') {
    // Method 1: text1, text2
    text1 = /** @type {string} */(a);
    diffs = this.diff_main(text1, /** @type {string} */(opt_b), true);
    if (diffs.length > 2) {
      this.diff_cleanupSemantic(diffs);
      this.diff_cleanupEfficiency(diffs);
    }
  } else if (a && typeof a == 'object' && typeof opt_b == 'undefined') {
    // Method 2: diffs
    diffs = /** @type {!Array.<!Array.<(number|string)>>} */(a);
    text1 = this.diff_text1(diffs);
  } else if (typeof a == 'string' && opt_b && typeof opt_b == 'object') {
    // Method 3: text1, diffs
    text1 = /** @type {string} */(a);
    diffs = /** @type {!Array.<!Array.<(number|string)>>} */(opt_b);
  } else {
    throw new Error('Unknown call format to patch_make.');
  }

  if (diffs.length === 0) {
    return [];  // Get rid of the null case.
  }
  var patches = [];
  var patch = new diff_match_patch.patch_obj();
  var patchDiffLength = 0;
  var char_count1 = 0;
  var char_count2 = 0;
  var prepatch_text = text1;
  var postpatch_text = text1;
  for (var i = 0; i < diffs.length; i++) {
    var diff_type = diffs[i][0];
    var diff_text = diffs[i][1];

    if (!patch.diffs.length && diff_type != DIFF_EQUAL) {
      // A new patch starts here.
      patch.start1 = char_count1;
      patch.start2 = char_count2;
    }

    switch (diff_type) {
      case DIFF_INSERT:
        patch.diffs.push(diffs[i]);
        patch.length2 += diff_text.length;
        postpatch_text = postpatch_text.substring(0, char_count2) + diff_text +
                         postpatch_text.substring(char_count2);
        break;
      case DIFF_DELETE:
        patch.length1 += diff_text.length;
        patch.diffs.push(diffs[i]);
        postpatch_text = postpatch_text.substring(0, char_count2) +
                         postpatch_text.substring(char_count2 +
                                                  diff_text.length);
        break;
      case DIFF_EQUAL:
        if (diff_text.length <= 2 * this.Patch_Margin &&
            patch.diffs.length !== 0 && diffs.length != i + 1) {
          // Small equality inside a patch.
          patch.diffs.push(diffs[i]);
          patch.length1 += diff_text.length;
          patch.length2 += diff_text.length;
        }

        if (diff_text.length >= 2 * this.Patch_Margin) {
          // Time for a new patch.
          if (patch.diffs.length) {
            this.patch_addContext_(patch, prepatch_text);
            patches.push(patch);
            patch = new diff_match_patch.patch_obj();
          }
          prepatch_text = postpatch_text;
          char_count1 = char_count2;
        }
        break;
    }

    // Update the current character count.
    if (diff_type !== DIFF_INSERT) {
      char_count1 += diff_text.length;
    }
    if (diff_type !== DIFF_DELETE) {
      char_count2 += diff_text.length;
    }
  }
  // Pick up the leftover patch if not empty.
  if (patch.diffs.length) {
    this.patch_addContext_(patch, prepatch_text);
    patches.push(patch);
  }

  return patches;
};


/**
 * Given an array of patches, return another array that is identical.
 * @param {!Array.<!diff_match_patch.patch_obj>} patches Array of Patch objects.
 * @return {!Array.<!diff_match_patch.patch_obj>} Array of Patch objects.
 */
diff_match_patch.prototype.patch_deepCopy = function(patches) {
  // Making a deep copy is not trivial in JavaScript.
  var patches_copy = [];
  for (var i = 0; i < patches.length; i++) {
    var patch = patches[i];
    var patch_copy = new diff_match_patch.patch_obj();
    patch_copy.diffs = [];
    for (var j = 0; j < patch.diffs.length; j++) {
      patch_copy.diffs[j] = patch.diffs[j].slice();
    }
    patch_copy.start1 = patch.start1;
    patch_copy.start2 = patch.start2;
    patch_copy.length1 = patch.length1;
    patch_copy.length2 = patch.length2;
    patches_copy[i] = patch_copy;
  }
  return patches_copy;
};


/**
 * Merge a set of patches onto the text.  Return a patched text, as well
 * as a list of true/false values indicating which patches were applied.
 * @param {!Array.<!diff_match_patch.patch_obj>} patches Array of Patch objects.
 * @param {string} text Old text.
 * @return {!Array.<string|!Array.<boolean>>} Two element Array, containing the
 *      new text and an array of boolean values.
 */
diff_match_patch.prototype.patch_apply = function(patches, text) {
  if (patches.length == 0) {
    return [text, []];
  }

  // Deep copy the patches so that no changes are made to originals.
  patches = this.patch_deepCopy(patches);

  var nullPadding = this.patch_addPadding(patches);
  text = nullPadding + text + nullPadding;

  this.patch_splitMax(patches);
  // delta keeps track of the offset between the expected and actual location
  // of the patch.  If there are patches expected to apply later in the
  // text, then a delta will be necessary to find the correct offset.
  var delta = 0;
  var results = [];
  for (var i = 0; i < patches.length; i++) {
    var expected_loc = patches[i].start2 + delta;
    var text1 = this.diff_text1(patches[i].diffs);
    var start_loc;
    var end_loc = -1;
    if (text1.length > this.Match_MaxBits) {
      // patch_apply#match_main determines if we should search for the patch
      // starting patch-apply#expected_loc, or if we should search from the
      // beginning of the text.
      start_loc = this.match_main(text, text1.substring(0, this.Match_MaxBits),
                                  expected_loc);
      if (start_loc != -1) {
        end_loc = this.match_main(text,
            text1.substring(text1.length - this.Match_MaxBits),
            expected_loc + text1.length - this.Match_MaxBits);
        if (start_loc == -1 || end_loc == -1) {
          // Did not find both ends of the patch.
          start_loc = -1;
        }
      }
    } else {
      start_loc = this.match_main(text, text1, expected_loc);
    }

    if (start_loc == -1) {
      // No match found.  :(
      results[i] = false;
      // Subtract the delta for this failed patch from subsequent patches.
      delta -= patches[i].length2 - patches[i].length1;
    } else {
      // Found a match.  :)
      results[i] = true;
      delta = start_loc - expected_loc;
      var text2;
      if (end_loc == -1) {
        text2 = text.substring(start_loc, start_loc + text1.length);
      } else {
        text2 = text.substring(start_loc, end_loc + this.Match_MaxBits);
      }
      if (text1 == text2) {
        // Perfect match.
        text = text.substring(0, start_loc) +
               this.diff_text2(patches[i].diffs) +
               text.substring(start_loc + text1.length);
      } else {
        // Imperfect match.
        // Run a diff to get a framework of equivalent indices.
        var diffs = this.diff_main(text1, text2, false);
        if (text1.length > this.Match_MaxBits &&
            this.diff_levenshtein(diffs) / text1.length >
            this.Patch_DeleteThreshold) {
          // The difference is too great to patch.
          results[i] = false;
        } else {
          this.diff_cleanupSemanticLossless(diffs);
          var index1 = 0;
          var j;
          for (j = 0; j < patches[i].diffs.length; j++) {
            var mod = patches[i].diffs[j];
            if (mod[0] !== DIFF_EQUAL) {
              var index2 = this.diff_xIndex(diffs, index1);
            }
            if (mod[0] === DIFF_INSERT) {  // Insertion
              text = text.substring(0, start_loc + index2) + mod[1] +
                     text.substring(start_loc + index2);
            } else if (mod[0] === DIFF_DELETE) {  // Deletion
              text = text.substring(0, start_loc + index2) +
                     text.substring(start_loc + this.diff_xIndex(diffs,
                         index1 + mod[1].length));
            }
            if (mod[0] !== DIFF_DELETE) {
              index1 += mod[1].length;
            }
          }
        }
      }
    }
  }
  // Strip the padding off.
  text = text.substring(nullPadding.length, text.length - nullPadding.length);
  return [text, results];
};


/**
 * Add some padding on text start and end so that edges can match something.
 * Intended to be called only from within patch_apply.
 * @param {!Array.<!diff_match_patch.patch_obj>} patches Array of Patch objects.
 * @return {string} The padding string added to each side.
 */
diff_match_patch.prototype.patch_addPadding = function(patches) {
  var paddingLength = this.Patch_Margin;
  var nullPadding = '';
  for (var x = 1; x <= paddingLength; x++) {
    nullPadding += String.fromCharCode(x);
  }

  // Bump all the patches forward.
  for (var i = 0; i < patches.length; i++) {
    patches[i].start1 += paddingLength;
    patches[i].start2 += paddingLength;
  }

  // Add some padding to provide context for patches.
  var patch = patches[0];
  var diffs = patch.diffs;
  if (diffs.length == 0 || diffs[0][0] != DIFF_EQUAL) {
    // Add nullPadding equality.
    diffs.unshift([DIFF_EQUAL, nullPadding]);
    patch.start1 -= paddingLength;  // Should be 0.
    patch.start2 -= paddingLength;  // Should be 0.
    patch.length1 += paddingLength;
    patch.length2 += paddingLength;
  } else if (paddingLength > diffs[0][1].length) {
    // Grow first equality.
    var extraLength = paddingLength - diffs[0][1].length;
    diffs[0][1] = nullPadding.substring(diffs[0][1].length) + diffs[0][1];
    patch.start1 -= extraLength;
    patch.start2 -= extraLength;
    patch.length1 += extraLength;
    patch.length2 += extraLength;
  }

  patch = patches[patches.length - 1];
  diffs = patch.diffs;
  if (diffs.length == 0 || diffs[diffs.length - 1][0] != DIFF_EQUAL) {
    // Add nullPadding equality.
    diffs.push([DIFF_EQUAL, nullPadding]);
    patch.length1 += paddingLength;
    patch.length2 += paddingLength;
  } else if (paddingLength > diffs[diffs.length - 1][1].length) {
    // Grow last equality.
    var extraLength = paddingLength - diffs[diffs.length - 1][1].length;
    diffs[diffs.length - 1][1] += nullPadding.substring(0, extraLength);
    patch.length1 += extraLength;
    patch.length2 += extraLength;
  }

  return nullPadding;
};


/**
 * Look through the patches and break up any which are longer than the maximum
 * limit of the match algorithm.
 * Intended to be called only from within patch_apply.
 * @param {!Array.<!diff_match_patch.patch_obj>} patches Array of Patch objects.
 */
diff_match_patch.prototype.patch_splitMax = function(patches) {
  var patch_size = this.Match_MaxBits;
  for (var i = 0; i < patches.length; i++) {
    if (patches[i].length1 <= patch_size) {
      continue;
    }
    var bigpatch = patches[i];
    // Remove the big patch to be replaced with smaller ones.
    patches.splice(i--, 1);
    var start1 = bigpatch.start1;
    var start2 = bigpatch.start2;
    var precontext = '';
    while (bigpatch.diffs.length !== 0) {
      // Create one smaller patch.
      var patch = new diff_match_patch.patch_obj();
      var empty = true;
      patch.start1 = start1 - precontext.length;
      patch.start2 = start2 - precontext.length;
      if (precontext !== '') {
        patch.length1 = patch.length2 = precontext.length;
        patch.diffs.push([DIFF_EQUAL, precontext]);
      }
      while (bigpatch.diffs.length !== 0 &&
             patch.length1 < patch_size - this.Patch_Margin) {
        var diff_type = bigpatch.diffs[0][0];
        var diff_text = bigpatch.diffs[0][1];
        if (diff_type === DIFF_INSERT) {
          // Insertions are harmless.
          patch.length2 += diff_text.length;
          start2 += diff_text.length;
          patch.diffs.push(bigpatch.diffs.shift());
          empty = false;
        } else if (diff_type === DIFF_DELETE && patch.diffs.length == 1 &&
                   patch.diffs[0][0] == DIFF_EQUAL &&
                   diff_text.length > 2 * patch_size) {
          // This is a large deletion.  Let it pass in one chunk.
          patch.length1 += diff_text.length;
          start1 += diff_text.length;
          empty = false;
          patch.diffs.push([diff_type, diff_text]);
          bigpatch.diffs.shift();
        } else {
          // Deletion or equality.  Only take as much as we can stomach.
          diff_text = diff_text.substring(0,
              patch_size - patch.length1 - this.Patch_Margin);
          patch.length1 += diff_text.length;
          start1 += diff_text.length;
          if (diff_type === DIFF_EQUAL) {
            patch.length2 += diff_text.length;
            start2 += diff_text.length;
          } else {
            empty = false;
          }
          patch.diffs.push([diff_type, diff_text]);
          if (diff_text == bigpatch.diffs[0][1]) {
            bigpatch.diffs.shift();
          } else {
            bigpatch.diffs[0][1] =
                bigpatch.diffs[0][1].substring(diff_text.length);
          }
        }
      }
      // Compute the head context for the next patch.
      precontext = this.diff_text2(patch.diffs);
      precontext =
          precontext.substring(precontext.length - this.Patch_Margin);
      // Append the end context for this patch.
      var postcontext = this.diff_text1(bigpatch.diffs)
                            .substring(0, this.Patch_Margin);
      if (postcontext !== '') {
        patch.length1 += postcontext.length;
        patch.length2 += postcontext.length;
        if (patch.diffs.length !== 0 &&
            patch.diffs[patch.diffs.length - 1][0] === DIFF_EQUAL) {
          patch.diffs[patch.diffs.length - 1][1] += postcontext;
        } else {
          patch.diffs.push([DIFF_EQUAL, postcontext]);
        }
      }
      if (!empty) {
        patches.splice(++i, 0, patch);
      }
    }
  }
};


/**
 * Add context to a patch list.
 * @param {!diff_match_patch.patch_obj} patch The patch to add context to.
 * @param {string} text The text being patched.
 * @private
 */
diff_match_patch.prototype.patch_addContext_ = function(patch, text) {
  if (text.length == 0) {
    return;
  }
  var pattern = text.substring(patch.start2, patch.start2 + patch.length1);
  var padding = 0;

  // Increase the context until it is unique.
  while (text.indexOf(pattern) !=
         text.lastIndexOf(pattern) &&
         pattern.length < this.Match_MaxBits - this.Patch_Margin -
         this.Patch_Margin) {
    padding += this.Patch_Margin;
    pattern = text.substring(Math.max(0, patch.start2 - padding),
                             patch.start2 + patch.length1 + padding);
  }
  // Add some padding for good measure.
  padding += this.Patch_Margin;

  // Add the prefix.
  var prefix = text.substring(Math.max(0, patch.start2 - padding),
                              patch.start2);
  if (prefix) {
    patch.diffs.unshift([DIFF_EQUAL, prefix]);
  }
  // Add the suffix.
  var suffix = text.substring(patch.start2 + patch.length1,
                              patch.start2 + patch.length1 + padding);
  if (suffix) {
    patch.diffs.push([DIFF_EQUAL, suffix]);
  }

  // Roll back the start points.
  patch.start1 -= prefix.length;
  patch.start2 -= prefix.length;
  // Extend the lengths.
  patch.length1 += prefix.length + suffix.length;
  patch.length2 += prefix.length + suffix.length;
};

diff_match_patch.prototype.diff_levenshtein = function(diffs) {
  var levenshtein = 0;
  var insertions = 0;
  var deletions = 0;
  for (var i = 0; i < diffs.length; i++) {
    var op = diffs[i][0];
    var data = diffs[i][1];
    switch (op) {
      case DIFF_INSERT:
        insertions += data.length;
        break;
      case DIFF_DELETE:
        deletions += data.length;
        break;
      case DIFF_EQUAL:
        levenshtein += Math.max(insertions, deletions);
        insertions = 0;
        deletions = 0;
        break;
    }
  }
  levenshtein += Math.max(insertions, deletions);
  return levenshtein;
};

diff_match_patch.prototype.diff_xIndex = function(diffs, loc) {
  var chars1 = 0;
  var chars2 = 0;
  var last_chars1 = 0;
  var last_chars2 = 0;
  var x;
  for (x = 0; x < diffs.length; x++) {
    if (diffs[x][0] !== DIFF_INSERT) {
      chars1 += diffs[x][1].length;
    }
    if (diffs[x][0] !== DIFF_DELETE) {
      chars2 += diffs[x][1].length;
    }
    if (chars1 > loc) {
      break;
    }
    last_chars1 = chars1;
    last_chars2 = chars2;
  }
  if (diffs.length != x && diffs[x][0] === DIFF_DELETE) {
    return last_chars2;
  }
  return last_chars2 + (loc - last_chars1);
};

diff_match_patch.prototype.diff_text1 = function(diffs) {
  var text = [];
  for (var i = 0; i < diffs.length; i++) {
    if (diffs[i][0] !== DIFF_INSERT) {
      text[i] = diffs[i][1];
    }
  }
  return text.join('');
};

diff_match_patch.prototype.diff_text2 = function(diffs) {
  var text = [];
  for (var i = 0; i < diffs.length; i++) {
    if (diffs[i][0] !== DIFF_DELETE) {
      text[i] = diffs[i][1];
    }
  }
  return text.join('');
};

diff_match_patch.prototype.match_main = function(text, pattern, loc) {
  loc = Math.max(0, Math.min(loc, text.length));
  if (text == pattern) {
    return 0;
  } else if (!text.length) {
    return -1;
  } else if (text.substring(loc, loc + pattern.length) == pattern) {
    return loc;
  } else {
    return this.match_bitap_(text, pattern, loc);
  }
};

diff_match_patch.prototype.match_bitap_ = function(text, pattern, loc) {
  if (pattern.length > this.Match_MaxBits) {
    throw new Error('Pattern too long for this browser.');
  }

  var s = this.match_alphabet_(pattern);
  var dmp = this;

  function match_bitapScore_(e, x) {
    var accuracy = e / pattern.length;
    var proximity = Math.abs(loc - x);
    if (!dmp.Match_Distance) {
      return proximity ? 1.0 : accuracy;
    }
    return accuracy + (proximity / dmp.Match_Distance);
  }

  var score_threshold = this.Match_Threshold;
  var best_loc = text.indexOf(pattern, loc);
  if (best_loc != -1) {
    score_threshold = Math.min(match_bitapScore_(0, best_loc), score_threshold);
    best_loc = text.lastIndexOf(pattern, loc + pattern.length);
    if (best_loc != -1) {
      score_threshold =
          Math.min(match_bitapScore_(0, best_loc), score_threshold);
    }
  }

  var matchmask = 1 << (pattern.length - 1);
  best_loc = -1;

  var bin_min, bin_mid;
  var bin_max = pattern.length + text.length;
  var last_rd;
  for (var d = 0; d < pattern.length; d++) {
    bin_min = 0;
    bin_mid = bin_max;
    while (bin_min < bin_mid) {
      if (match_bitapScore_(d, loc + bin_mid) <= score_threshold) {
        bin_min = bin_mid;
      } else {
        bin_max = bin_mid;
      }
      bin_mid = Math.floor((bin_max - bin_min) / 2 + bin_min);
    }
    bin_max = bin_mid;
    var start = Math.max(1, loc - bin_mid + 1);
    var finish = Math.min(loc + bin_mid, text.length) + pattern.length;

    var rd = Array(finish + 2);
    rd[finish + 1] = (1 << d) - 1;
    for (var j = finish; j >= start; j--) {
      var charMatch;
      if (text.length <= j - 1 || !s.hasOwnProperty(text.charAt(j - 1))) {
        charMatch = 0;
      } else {
        charMatch = s[text.charAt(j-1)];
      }

      if (d === 0) {
        rd[j] = ((rd[j + 1] << 1) | 1) & charMatch;
      } else {
        rd[j] = (((rd[j + 1] << 1) | 1) & charMatch) |
                (((last_rd[j + 1] | last_rd[j]) << 1) | 1) |
                last_rd[j + 1];
      }
      if (rd[j] & matchmask) {
        var score = match_bitapScore_(d, j - 1);
        if (score <= score_threshold) {
          score_threshold = score;
          best_loc = j - 1;
          if (best_loc > loc) {
            start = Math.max(1, 2 * loc - best_loc);
          } else {
            break;
          }
        }
      }
    }
    if (match_bitapScore_(d + 1, loc) > score_threshold) {
      break;
    }
    last_rd = rd;
  }
  return best_loc;
};

diff_match_patch.prototype.match_alphabet_ = function(pattern) {
  var s = {};
  for (var i = 0; i < pattern.length; i++) {
    s[pattern.charAt(i)] = 0;
  }
  for (var i = 0; i < pattern.length; i++) {
    s[pattern.charAt(i)] |= 1 << (pattern.length - i - 1);
  }
  return s;
};

// --- END of Diff-Match-Patch Library ---


// --- HELPER FUNCTIONS ---
async function calculateHash(text) {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  return hashHex;
}
function sanitizePathForDirName(path) {
  let sanitized = path.replace(/[\/\\]/g, '__');
  sanitized = sanitized.replace(/[:*?"<>|]/g, '');
  return sanitized;
}

// Instantiate the diffing engine
const dmp = new diff_match_patch();
// Cache for reconstructed file versions to improve performance
const rebuildCache = new Map();

function GitControl() {
  // ====================================================================
  //                        -- YOUR SETTINGS --
  const fileName = "GitControl.component.v2.md";
  // ====================================================================

  const safeDirName = sanitizePathForDirName(fileName);
  const fileGitBaseDir = `.datacore/.git/${safeDirName}`;
  const objectsDir = `${fileGitBaseDir}/objects`;
  const headFile = `${fileGitBaseDir}/HEAD`;

  const [codeBlocks, setCodeBlocks] = useState([]);
  const [activeTabIndex, setActiveTabIndex] = useState(0);
  const [statusMessage, setStatusMessage] = useState("Initializing version control...");

  /**
   * Reconstructs a file's full content from a given hash by walking
   * backwards through parent commits and applying patches.
   */
  const rebuildFileFromHistory = async (targetHash) => {
    if (rebuildCache.has(targetHash)) {
      return rebuildCache.get(targetHash);
    }
    
    const adapter = app.vault.adapter;
    const objectPath = `${objectsDir}/${targetHash}.json`;
    if (!(await adapter.exists(objectPath))) {
        throw new Error(`Corrupt history: Cannot find object ${targetHash}`);
    }

    const commitObject = JSON.parse(await adapter.read(objectPath));

    // Base case: This is the initial commit, which contains the full content.
    if (commitObject.content !== undefined) {
      rebuildCache.set(targetHash, commitObject.content);
      return commitObject.content;
    }

    // Recursive step: This is a patch. Get the parent's content first.
    if (commitObject.parent && commitObject.patch) {
      const parentContent = await rebuildFileFromHistory(commitObject.parent);
      const patches = dmp.patch_fromText(commitObject.patch);
      const [rebuiltContent, results] = dmp.patch_apply(patches, parentContent);
      
      // Check if the patch applied cleanly
      if (results.some(applied => !applied)) {
          throw new Error(`Failed to apply patch for commit ${targetHash}`);
      }
      
      rebuildCache.set(targetHash, rebuiltContent);
      return rebuiltContent;
    }
    
    throw new Error(`Invalid commit object structure for ${targetHash}`);
  };

  useEffect(() => {
    const processFileAndCommitChanges = async () => {
      try {
        rebuildCache.clear(); // Clear cache on each run
        const adapter = app.vault.adapter;
        const file = app.vault.getAbstractFileByPath(fileName);
        if (!file) {
          setStatusMessage(`Error: Source file not found.`);
          setCodeBlocks([]);
          return;
        }
        const fullFileContent = await app.vault.read(file);
        const currentHash = await calculateHash(fullFileContent);
        
        // --- VERSIONING LOGIC (Now with Diffing) ---
        const objectPath = `${objectsDir}/${currentHash}.json`;

        if (await adapter.exists(objectPath)) {
          setStatusMessage(`No changes detected. (v: ${currentHash.slice(0, 7)})`);
        } else {
          setStatusMessage(`New version found. Committing...`);
          
          await adapter.mkdir(fileGitBaseDir).catch(()=>{});
          await adapter.mkdir(objectsDir).catch(()=>{});
          
          let commitObject;
          const headExists = await adapter.exists(headFile);

          if (!headExists) {
            // This is the initial commit. Store the full file content.
            commitObject = {
              source: fileName,
              timestamp: new Date().toISOString(),
              content: fullFileContent, // Full content
            };
            setStatusMessage(`Initial commit created: v: ${currentHash.slice(0, 7)}`);
          } else {
            // This is a subsequent commit. Store a patch.
            const previousHash = await adapter.read(headFile);
            const previousContent = await rebuildFileFromHistory(previousHash);

            const patches = dmp.patch_make(previousContent, fullFileContent);
            const patchText = dmp.patch_toText(patches);

            commitObject = {
              source: fileName,
              timestamp: new Date().toISOString(),
              parent: previousHash, // Link to the previous version
              patch: patchText,      // Store only the changes
            };
            setStatusMessage(`Successfully committed v: ${currentHash.slice(0, 7)}`);
          }

          await adapter.write(objectPath, JSON.stringify(commitObject, null, 2));
          await adapter.write(headFile, currentHash);
        }

        // --- PARSING & DISPLAY LOGIC (Unchanged) ---
        const regex = /(?:^# (.*))|(?:^```([^\r\n]*)\r?\n([\s\S]*?)\r?\n```)/gm;
        const allMatches = [...fullFileContent.matchAll(regex)];
        const parsedBlocks = [];
        let lastHeader = null;
        let untitledBlockCounter = 1;

        for (const match of allMatches) {
            const headerMatch = match[1];
            const codeMatch = match[3];
            if (headerMatch !== undefined) {
                lastHeader = headerMatch.trim();
            } else if (codeMatch !== undefined) {
                const title = lastHeader ? lastHeader : `Block ${untitledBlockCounter++}`;
                parsedBlocks.push({ title: title, content: codeMatch.trim() });
                lastHeader = null;
            }
        }
        
        if (parsedBlocks.length === 0) {
          setStatusMessage(prev => `${prev} | Note: No code blocks were found.`);
          setCodeBlocks([]);
        } else {
          setCodeBlocks(parsedBlocks);
          setActiveTabIndex(0);
        }

      } catch (error) {
        console.error("GitControl Critical Error:", error);
        setStatusMessage(`A critical error occurred: ${error.message}`);
        setCodeBlocks([{ title: "Error", content: `// ${error.message}` }]);
      }
    };

    processFileAndCommitChanges();
    
  }, [fileName]);

  // --- RENDER LOGIC with TABS (Unchanged) ---
  return (
    <dc.Stack style={{ padding: "10px 15px", fontFamily: "monospace" }}>
      <div style={{ padding: "5px 10px", marginBottom: "10px", background: "#eef", border: "1px solid #cce", borderRadius: "4px", fontSize: "0.9em", color: "#335" }}>
          Status: {statusMessage}
      </div>
      {codeBlocks.length > 0 && (
          <div style={{ display: 'flex', borderBottom: '1px solid #ddd', flexWrap: 'wrap', marginBottom: '10px' }}>
              {codeBlocks.map((block, index) => (
                  <button
                      key={`${block.title}-${index}`}
                      onClick={() => setActiveTabIndex(index)}
                      style={{ padding: '8px 12px', cursor: 'pointer', border: 'none', borderBottom: index === activeTabIndex ? '3px solid #007acc' : '3px solid transparent', background: index === activeTabIndex ? '#f0f8ff' : 'none', fontWeight: index === activeTabIndex ? 'bold' : 'normal', color: 'inherit', fontSize: '0.9em' }}>
                      {block.title}
                  </button>
              ))}
          </div>
      )}
      <pre style={{ background: '#f4f4f4', border: '1px solid #ddd', padding: '1em', borderRadius: '5px', whiteSpace: 'pre-wrap', wordBreak: 'break-word', color: '#333', margin: "0" }}>
        <code>
          {codeBlocks.length > 0 && codeBlocks[activeTabIndex]
            ? codeBlocks[activeTabIndex].content
            : '// No code blocks were found in the file.'}
        </code>
      </pre>
    </dc.Stack>
  );
}

return { GitControl };
```


# Test

```jsx
const { WorldView } = await dc.require(dc.headerLink("B25MovingCat.component.v3.md", "ViewComponent"));
return <WorldView />;

```