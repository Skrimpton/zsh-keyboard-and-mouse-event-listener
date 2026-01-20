# This is a hob-gobbled together event-listener for ZSH (with many functions not doing anything 
https://private-user-images.githubusercontent.com/64572787/537805139-aef01acd-53b5-4702-86c7-2ceb6374db06.mp4?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Njg4NzU3OTEsIm5iZiI6MTc2ODg3NTQ5MSwicGF0aCI6Ii82NDU3Mjc4Ny81Mzc4MDUxMzktYWVmMDFhY2QtNTNiNS00NzAyLTg2YzctMmNlYjYzNzRkYjA2Lm1wND9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNjAxMjAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjYwMTIwVDAyMTgxMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTUwZjlhZWQ0M2FjODE4MGYxYTRjZDVjODlmOTBjYzE1ZmU4OWY0OWI2NDhhZWJlOTliYWUyYzgyMTdiOWRiMDYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.IyvmgrPM49aXVqvWKFMd87hAdT3FqiP2bQQWcFPeX68
###### written by what is still an enthusiastic, but full-on novice and moron<br>assisted by the very helpful dystopian nightmare and frequent confident liar: Copilot
## Use it to build (bad) VIM-like GUI's (at least you don't have to deal with zcurses)<br>…or catch presses without calling a new private buffer.<br>The options are limited, but some!
##### Tested on: Konsole v23.08.5 - zsh 5.9 (x86_64-pc-linux-gnu) | Termux 0.118.3 - zsh 5.9 (aarch64-unknown-linux-android): 
#### Termux notes: 
- Drag doesn't work at all.
- Keys, scroll and left "click" are caught. 
- Every time finger lands it's a \<MOUSE_LEFT\> press and release.
- \<DELETE\> is different: \<BACKSPACE\> is the same
### There is no guarantee this will work in your environment (neither flawlessly nor at all)<br>DO NOT RUN IF YOU CAN'T EXIT AND REBOOT SESSION!  
#### …just in case, these are friends to have in times of need:
```zsh
reset
```
```zsh
tput reset
```
## THE SCRIPT:

```zsh
#!/usr/bin/env zsh

zmodload zsh/terminfo
zmodload zsh/datetime
setopt typeset_silent
setopt extended_glob

readonly x_BASE_DEBOUNCE=0.23
readonly x_MIN_WIDTH=10
readonly x_MIN_HEIGHT=10
readonly x_DEFAULT_CURSOR="│"

#  INTERNAL HACK(S) / CONSTANTS
# readonly x_ZERO_WIDTH_SPACE="​" # U+200B
# readonly x_S2="${x_ZERO_WIDTH_SPACE}" # Internal Command Separator
# readonly x_ZERO_WIDTH_NON_JOINER="‌"  # U+200C
# readonly x_S1="${x_ZERO_WIDTH_NON_JOINER}"
# readonly x_ADD="${x_S1}add"
# readonly x_GT="${x_S1}gt"
# readonly x_LT="${x_S1}lt"
# readonly x_NOT="${x_S1}not"
# readonly x_NORM="${x_S2}"


float   x_MOUSE_CLICK_THRESHOLD_SIM=0.10
float   x_MOUSE_CLICK_THRESHOLD_DBL=$x_BASE_DEBOUNCE
float   x_MOUSE_DBL_CLICK_DIFF=0.0
integer x_MOUSE_DBL_CLICK=0
integer x_MOUSE_CLICK_SWITCH=0
typeset x_MOUSE_COMBOSYM
integer x_LAST_MOUSE_CODE=-1
typeset x_LAST_MOUSE_SYM="NaN"
typeset -a x_BUTTON_ORDER=()


integer x_DEBUG=1

# GLOBAL MUTABLES
x_ORIG_STTY=""
x_TIMESTAMP_KEY=$EPOCHREALTIME # TODO #1) float: dropped events and inaccuracy; why? | (typeset): accurate enough, given x_MOUSE_CLICK_THRESHOLD_DBL is tuned for terminal
x_TIMESTAMP_CLICK_LEFT=$EPOCHREALTIME # TODO #1
x_TIMESTAMP_CLICK_MIDDLE=$EPOCHREALTIME # TODO #1
x_TIMESTAMP_CLICK_RIGHT=$EPOCHREALTIME # TODO #1

# EVENT MAP
x_EVENT=""
x_EVENT_MOUSE=""

# ### GUI ELEMENTS

# ESCAPE CODE HELPERS
e_R=$'\r' # printf easy identifier: return cursor to start of line
e_E=$'\033[0m' # end mod
e_B=$'\033[1m' # bold
e_D=$'\033[2m' # dim
e_U=$'\033[4m' # underline
e_I=$'\033[7m' # invert
e_CL=$'\033[2K' # clear line
e_MTR=$'\033[%d;1H' # %d is a placeholder for a digit
# BG
bg_GRAY=$'\033[48;5;235m'
bg_BLUE=$'\033[48;5;17m'
bg_HL=$'\033[48;5;89m'
# FG
fg_RED=$'\033[38;5;196m'
fg_PINK=$'\033[38;5;176m'
fg_BLUE=$'\033[38;5;39m'
fg_WHITE=$'\033[38;5;255m'
fg_GREEN=$'\033[38;5;40m'
fg_GREEN2=$'\033[38;5;157m'
fg_YELLOW=$'\033[38;5;226m'
fg_HL=$'\033[38;5;255m'


# typeset -A x_GUI=(
#   prompt_cmd "⋯"
#   prompt_inp "≫"
#   prompt_src "❓"
#   prompt_sep ""
# )

#  INIT, EXIT & TRAP
xCleanup() {
  clear
  xScreen OldScreen \
  MousePress_OFF \
  MouseExtended_OFF \
  MouseDrag_OFF \
  MouseMotion_OFF \
  Cursor_SHOW
  [[ -n $x_ORIG_STTY ]] && stty "$x_ORIG_STTY"
}

xInit() { local x
  local -a timestamps=(ClickLeft ClickMid ClickRight Key)
  xScreen NewScreen \
  MousePress_ON \
  MouseExtended_ON \
  MouseDrag_ON \
  MouseSGR_ON \
  MouseMotion_ON \
  Cursor_HIDE


  clear
  x_ORIG_STTY=$(stty -g)
  stty -echo -icanon time 0 min 1
  stty susp undef
  for x in "${timestamps[@]}"; do xTimestamp "$x"; done
}

xTrap() {
  local sig="$1" code
  case "$sig" in
    HUP)  xCleanup; code=129 ;; INT)  xCleanup; code=130 ;;
    QUIT) xCleanup; code=131 ;; TERM) xCleanup; code=143 ;;
    ERR)  xCleanup; code=1 ;; # general error
    *)    code=0 ;;
  esac
  exit $code
}

TRAPINT()  { xTrap INT }
TRAPTERM() { xTrap TERM }
TRAPHUP()  { xTrap HUP }
TRAPQUIT() { xTrap QUIT }


#  GENERIC HELPERS

zZ() (:)  # VIP: Refreshes $COLUMNS, $LINES, let's TRAPWINCH() function

xScreen() {
  while (( $# )); do
    case "$1" in
      Cursor_SHOW) printf $'\033[?25h';;
      Cursor_HIDE) printf $'\033[?25l';;
      Cursor_SAVE) printf $'\0337';;
      Cursor_RESTORE) printf $'\0338';;
      MouseSGR_ON) printf '\e[>1u';;
      MousePress_ON) printf $'\033[?1000h';;
      MousePress_OFF) printf $'\033[?1000l';;
      MouseDrag_ON) printf $'\033[?1002h';;
      MouseDrag_OFF) printf $'\033[?1002l';;
      MouseMotion_ON) printf $'\033[?1003h';;
      MouseMotion_OFF) printf $'\033[?1003l';;
      MouseExtended_ON) printf $'\033[?1006h';;
      MouseExtended_OFF) printf $'\033[?1006l';;
      NewScreen) printf $'\033[?1049h'; sleep 0.001 ;;
      OldScreen) printf $'\033[?1049l';;
      ClearFromCursorUp) printf $'\033[1J\033[H';;
    esac
    shift
  done
}

xPrint() {
  case "$1" in
    clearline) shift
      printf "%b%s" "${e_R}${e_E}${e_CL}" "$1"
    ;;
    reset_line_at_row) shift; [[ -z $1 ]] && return
      local row="$1"
      printf "${e_MTR}${e_CL}\r${e_E}" "$row"
    ;;
    atline) shift; [[ -z $1 ]] && return
      local row="$1"
      printf "${e_MTR}${e_CL}\r${e_E}%s" "$row" "$2"
    ;;
  esac
}

xIsNumeric() { local arg="$1" mode="${2:-any}" num
  # Usage:   xIsNumeric <value> [mode]
  # mode:
  #   "any"      : any number (default)
  #   "positive" : must be > 0
  #   "negative" : must be < 0
  # return: 0 = valid | 1 = invalid

  arg="${arg//[[:space:]]/}"

  [[ -z "$arg" ]] && return 1
  # Examples accepted:  12  |  -12  |  +12.5  |  .5  |  5.  |  -3.14e10  |  +2.1E-3
  [[ $arg =~ '^[+-]?((([0-9]+([.][0-9]*)?)|([.][0-9]+))([eE][+-]?[0-9]+)?)$' ]] || return 1

  num=$(( arg + 0.0 ))   # force float interpretation
  case "$mode" in
    pos|positive) (( num > 0 )) || return 1 ;;
    neg|negative) (( num < 0 )) || return 1 ;;
    *) return 2 ;; # invalid mode
  esac
  return 0
}

# DEBOUNCE

xTimestamp() { local now=$EPOCHREALTIME
  case "$1" in
    Key)        x_TIMESTAMP_KEY=$now ;;
    ClickLEFT)  x_TIMESTAMP_CLICK_LEFT=$now ;;
    ClickMIDDLE)   x_TIMESTAMP_CLICK_MIDDLE=$now ;;
    ClickRIGHT) x_TIMESTAMP_CLICK_RIGHT=$now ;;
  esac
}

xDelta() { local last=$1 n=$EPOCHREALTIME diff_ms
  x_MOUSE_DBL_CLICK_DIFF=$(( ((n - last) * 10000) / 10000.0 ))
}

xDebounce() { local delta=$1 th=$2
  (( delta < th )) && return 0  # debounce (reject)
  return 1
}

xDebounceKey() { local r=1 th=$x_BASE_DEBOUNCE last=$x_TIMESTAMP_KEY
  local delta=$(xDelta "$last")
  xDebounce "$delta" "$th" && r=0
  xTimestamp Key
  return $r
}

xDebounceMouse() { local btn="$1" th=$x_MOUSE_CLICK_THRESHOLD_DBL last; integer r=1
  case "$btn" in
    LEFT) last=$x_TIMESTAMP_CLICK_LEFT ;;
    MIDDLE) last=$x_TIMESTAMP_CLICK_MIDDLE ;;
    RIGHT) last=$x_TIMESTAMP_CLICK_RIGHT ;;
    *) return 1 ;;
  esac
  xDelta "$last"
  xDebounce $x_MOUSE_DBL_CLICK_DIFF "$th" && { r=0; x_MOUSE_DBL_CLICK=1; }  # debounced
  xTimestamp "Click$btn"
  return $r
}
xDebounceCheckPrevious() { local btn="$1" n=$2 last diff th=$x_MOUSE_CLICK_THRESHOLD_SIM; integer r=1
  case "$btn" in
    LEFT) last=$x_TIMESTAMP_CLICK_LEFT ;;
    MIDDLE) last=$x_TIMESTAMP_CLICK_MIDDLE ;;
    RIGHT) last=$x_TIMESTAMP_CLICK_RIGHT ;;
    *) return 1 ;;
  esac
  diff=$(( ((n - last) * 10000) / 10000.0 ))
  xDebounce $diff $th && r=0  # debounced
  return $r
}


# ### HANDLE PARSED EVENTS BEGIN

xHandleEvent() { local raw="$1" e="$2"
  x_EVENTRAW="$raw"
  x_EVENT="$e"

  xPrint atline 2 "${(q)raw} → $e"

  case "$e" in
    # mouse
    MOUSE_SCROLL_UP*)   ;;
    MOUSE_SCROLL_DOWN*) ;;
    MOUSE_*_PRESS*)     xMousePress "$e" ;;
    MOUSE_*_RELEASE*)   xMouseRelease "$e" ;;
    MOUSE_*_MOTION*)    xPrint atline 2 "$e" ;;
  esac

  case "$e" in
    CTRL_C|CHAR:q)  xTrap INT ;;
    KEY_UP*) xPrint atline 2 "${(q)raw} → <xHandleEvent:KEY-UP> $e" ;;
  esac
}

xMouseDoubleClick() { local btn="$1" sym=$2 code=$3 ex=$4 ey=$5
  xPrint atline 2 "DOUBLE $btn $code $ex $ey"
}
xMousePress(){ local e="$1" a sym ba lba btn lbtn ds=0 now
  a=("${(s: :)e}")
  sym="${a[1]}"
  ba=(${(s:_:)sym})
  btn="${ba[2]}"

  xDebounceMouse "$btn"; integer debounced=$?
  now=$EPOCHREALTIME
  integer code="${a[2]}" ex="${a[3]}" ey="${a[4]}"

  lbtn="${x_BUTTON_ORDER[-1]:=$btn}"
  (( ${x_BUTTON_ORDER[(Ie)$btn]} == 0 )) && x_BUTTON_ORDER+=("$btn")
  x_MOUSE_CLICK_SWITCH=0

  (( x_LAST_MOUSE_CODE != code )) && { x_MOUSE_DBL_CLICK=0; x_MOUSE_CLICK_SWITCH=1; }
  if (( debounced == 0 ))
  then xMouseDoubleClick "$btn" "$sym" $code $ex $ey
  else
    if [[ $lbtn == $btn ]]
    then xPrint atline 2 "<$btn> $sym $code $ex $ey"
#     elif xDebounceCheckPrevious "$lbtn" $now
    else xPrint atline 2 "<$btn> $sym $code $ex $ey ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"
    fi
  fi
  (( x_LAST_MOUSE_CODE = code ))
}

xMouseRelease(){ local e="$1" a sym ba btn mod bm
  a=("${(s: :)e}")
  sym="${a[1]}"
  integer code="${a[2]}" ex="${a[3]}" ey="${a[4]}"
  ba=(${(s:_:)sym})
  btn="${ba[2]}"
  mod=${e##*RELEASE}
  mod=${mod%% *}
  bm="${btn}${mod}"
#   case $mod in
#     _ALT) xPrint clearline "_ALT) RELEASE $bm $code $ex $ey";;
#     _CTRL) xPrint clearline "_CTRL) RELEASE $bm $code $ex $ey";;
#     *)
#   esac
  xPrint atline 2 "RELEASE $e"
#   xPrint atline 2 "RELEASE $sym $code $ex $ey ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"
  x_MOUSE_DBL_CLICK=0
}

# ### HANDLE PARSED EVENTS END


#  MOUSE PARSING

xParseMouseSym() { local l="$1" c="$2" t="$3" s="$4"
  # RELEASE (SGR: 'm')
  [[ "$l" == "m" ]] && {
    case $(( c & 3 )) in
      0) x_BUTTON_ORDER=(${x_BUTTON_ORDER:#LEFT}); x_EVENT_MOUSE="MOUSE_LEFT_RELEASE${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
      1) x_BUTTON_ORDER=(${x_BUTTON_ORDER:#MIDDLE}); x_EVENT_MOUSE="MOUSE_MIDDLE_RELEASE${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
      2) x_BUTTON_ORDER=(${x_BUTTON_ORDER:#RIGHT}); x_EVENT_MOUSE="MOUSE_RIGHT_RELEASE${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
      *) x_BUTTON_ORDER=(${x_BUTTON_ORDER:#UNKNOWN*}); x_EVENT_MOUSE="MOUSE_UNKNOWN_RELEASE${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
    esac
  }

  # SCROLL
  case "$c" in
    64) x_EVENT_MOUSE="MOUSE_SCROLL_UP ${s}"; return ;;
    65) x_EVENT_MOUSE="MOUSE_SCROLL_DOWN ${s}"; return ;;
    80) x_EVENT_MOUSE="MOUSE_CTRL_SCROLL_UP ${s}"; return ;;
    81) x_EVENT_MOUSE="MOUSE_CTRL_SCROLL_DOWN ${s}"; return ;;
  esac
  # DRAG / MOTION (bit 32)
  if (( c & 32 )); then
    local base=$(( c & 3 ))
    case "$base" in
      0) x_EVENT_MOUSE="MOUSE_LEFT${t}_DRAG ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
      1) x_EVENT_MOUSE="MOUSE_MIDDLE${t}_DRAG ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
      2) x_EVENT_MOUSE="MOUSE_RIGHT${t}_DRAG ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
      3) x_EVENT_MOUSE="MOUSE_MOTION${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;  # no button pressed
      *) x_EVENT_MOUSE="MOUSE_UNKNOWN_DRAG${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
    esac
  fi

  # PRESS
  case $(( c & 3 )) in
    0) x_EVENT_MOUSE="MOUSE_LEFT_PRESS${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
    1) x_EVENT_MOUSE="MOUSE_MIDDLE_PRESS${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
    2) x_EVENT_MOUSE="MOUSE_RIGHT_PRESS${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
    *) x_EVENT_MOUSE="MOUSE_UNKNOWN_PRESS${t} ${s} ${x_BUTTON_ORDER[*]:+"<$x_BUTTON_ORDER[*]>"}"; return ;;
  esac

  x_EVENT_MOUSE="MOUSE_UNKNOWN ${s}"
}


xParseMouse() { local mod modb type state seq
  # Detect prefix
  case "$1" in
    $'\e[<'*) seq="${1#$'\e[<'}" ;;   # SGR
    $'\e['*)  seq="${1#$'\e['}" ;;    # urxvt
    *) return ;;
  esac

  # Must contain at least 3 semicolon-separated fields
  [[ "$seq" != *";"*";"* ]] && return

  # C ; X ; Y M/m
  local c="${seq%%;*}" rest="${seq#*;}"
  local x="${rest%%;*}" y="${rest#*;}"
  local last="${y: -1}"   # M or m
  y="${y%?}"

  # Validate numeric fields
  [[ "$c" =~ ^[0-9]+$ ]] || return
  [[ "$x" =~ ^[0-9]+$ ]] || return
  [[ "$y" =~ ^[0-9]+$ ]] || return

  integer code=$c ex=$x ey=$y

  case "$code" in
    8|9|10|40|41|42)      mod="_ALT" ;;
    16|17|18|48|49|50)    mod="_CTRL" ;;
    24|25|26|56|57|58)    mod="_CTRL_ALT" ;;
  esac

  (( x_MOUSE_DBL_CLICK )) && modb="_DBL_CLICK"
  type="${modb}${mod}"
  state="$code $ex $ey"

  xParseMouseSym "$last" "$code" "$type" "$state"
  xHandleEvent "$seq" "$x_EVENT_MOUSE"
  x_EVENT_MOUSE=""
}

xParseKey() { local seq="$1" e="UNKNOWN"; [[ -z "$seq" ]] && { xHandleEvent "$seq" "$e"; return; }
  e="${e}:$seg"
  case "$seq" in # printable or basic control

    $'\b')                   e="KEY_BACKSPACE" ;; #|$'\x7f'|$'\x08'|$'\177'
    $'\n'|$'\r')             e="KEY_ENTER" ;;
    $'\t')                   e="KEY_TAB" ;;
    $'\001')                 e="CTRL_A" ;;
    $'\002')                 e="CTRL_B" ;;
    $'\003')                 e="CTRL_C" ;;
    $'\004')                 e="CTRL_D" ;;
    $'\005')                 e="CTRL_E" ;;
    $'\006')                 e="CTRL_F" ;;
    $'\a')                   e="CTRL_G" ;;
    $'\v')                   e="CTRL_K" ;;
    $'\f')                   e="CTRL_L" ;;
    $'\016')                 e="CTRL_N" ;;
    $'\017')                 e="CTRL_O" ;;
    $'\020')                 e="CTRL_P" ;;
    $'\022')                 e="CTRL_R" ;;
    $'\024')                 e="CTRL_T" ;;
    $'\025')                 e="CTRL_U" ;;
    $'\026')                 e="CTRL_V" ;;
    $'\027')                 e="CTRL_W" ;;
    $'\030')                 e="CTRL_X" ;;
    $'\031')                 e="CTRL_Y" ;;
    $'\032')                 e="CTRL_Z" ;;
    $'\035')                 e="CTRL_5" ;;
    $'\036')                 e="CTRL_6" ;;
    $'\037')                 e="CTRL_7" ;;
    $'\177')                 e="CTRL_8" ;;
    $'\0')                   e="CTRL_SPACE" ;;
    $'\e') e="KEY_ESC" ;;

    # simple arrows
    $'\e[A') e="KEY_UP" ;;
    $'\e[B') e="KEY_DOWN" ;;
    $'\e[C') e="KEY_RIGHT" ;;
    $'\e[D') e="KEY_LEFT" ;;

    # Home / End
    $'\e[H') e="KEY_HOME" ;;
    $'\e[F') e="KEY_END"  ;;

    # tilde-terminated keys
    $'\e[2~')  e="KEY_INSERT" ;;
    $'\e[3~')  e="KEY_DELETE" ;;
    $'\e[5~')  e="KEY_PGUP" ;;
    $'\e[6~')  e="KEY_PGDN" ;;
    $'\e[15~') e="KEY_F5" ;;
    $'\e[17~') e="KEY_F6" ;;
    $'\e[18~') e="KEY_F7" ;;
    $'\e[19~') e="KEY_F8" ;;
    $'\e[20~') e="KEY_F9" ;;
    $'\e[21~') e="KEY_F10" ;;
    $'\e[24~') e="KEY_F12" ;;

    # Modified arrows (Ctrl/Shift/Alt combos)
    $'\e[1;5A') e="CTRL_UP" ;;
    $'\e[1;5B') e="CTRL_DOWN" ;;
    $'\e[1;5C') e="CTRL_RIGHT" ;;
    $'\e[1;5D') e="CTRL_LEFT" ;;

    $'\e[1;2A') e="SHIFT_UP" ;;
    $'\e[1;2B') e="SHIFT_DOWN" ;;
    $'\e[1;2C') e="SHIFT_RIGHT" ;;
    $'\e[1;2D') e="SHIFT_LEFT" ;;

    $'\e[1;3A') e="ALT_UP" ;;
    $'\e[1;3B') e="ALT_DOWN" ;;
    $'\e[1;3C') e="ALT_RIGHT" ;;
    $'\e[1;3D') e="ALT_LEFT" ;;

    $'\e[1;4A') e="ALT_SHIFT_UP" ;;
    $'\e[1;4B') e="ALT_SHIFT_DOWN" ;;
    $'\e[1;4C') e="ALT_SHIFT_RIGHT" ;;
    $'\e[1;4D') e="ALT_SHIFT_LEFT" ;;

    $'\e[1;6A') e="CTRL_SHIFT_UP" ;;
    $'\e[1;6B') e="CTRL_SHIFT_DOWN" ;;
    $'\e[1;6C') e="CTRL_SHIFT_RIGHT" ;;
    $'\e[1;6D') e="CTRL_SHIFT_LEFT" ;;

    $'\e[1;7A') e="CTRL_ALT_UP" ;;
    $'\e[1;7B') e="CTRL_ALT_DOWN" ;;
    $'\e[1;7C') e="CTRL_ALT_RIGHT" ;;
    $'\e[1;7D') e="CTRL_ALT_LEFT" ;;

    $'\e[1;8A') e="CTRL_ALT_SHIFT_UP" ;;
    $'\e[1;8B') e="CTRL_ALT_SHIFT_DOWN" ;;
    $'\e[1;8C') e="CTRL_ALT_SHIFT_RIGHT" ;;
    $'\e[1;8D') e="CTRL_ALT_SHIFT_LEFT" ;;

    *) if [[ $seq == *[^[:print:]]* ]]; then
        e="NON_PRINTABLE"
        [[ $seq == $'\033'* ]] && {
          local printcheck=${seq/$'\033'/}
          case "$printcheck" in
            ' ') e="ALT_SPACE" ;;
            *) [[ -n $printcheck ]] && e="ALT_${printcheck}" ;;
          esac
        }
      else e="CHAR:$seq"
      fi
  esac
  xHandleEvent "$seq" "$e"
}

xRawPreparse() { local buf="$1" seq e # raw
  # takes $1 = raw buffer, may contain several events
  while  [[ -n "$buf" ]]; do
    if   [[ "$buf" =~ $'^\e\\[<[0-9]+;[0-9]+;[0-9]+[Mm]' ]]
    then seq="${buf:0:${#MATCH}}"
    elif [[ "$buf" =~ $'^\e\\[[^[:alpha:]~u]*[[:alpha:]~u]' ]]
    then seq="${buf:0:${#MATCH}}"
    elif [[ "$buf" =~ $'^\e.$' ]]
    then seq="${buf:0:2}"
    else seq="${buf[1,1]}"
    fi;  [[ -z "$seq" ]] && seq="$buf"

    buf="${buf#$seq}" # remove the processed seq from the front of buf

    if   [[ "$seq" == $'\e[<'* || "$seq" == $'\e['[0-9]##';'[0-9]##';'[0-9]##[Mm] ]] # mouse: SGR or urxvt
    then xParseMouse "$seq"
    else xParseKey "$seq"
    fi
  done
}

xRawRead() { local buf="" chunk=""
  read -sk1 buf -t 0.05 || return 1 # read first byte or do nothing
  while read -t 0 -sk1 chunk; do buf+="$chunk"; done
  xRawPreparse "$buf"
}

#  DEMO EVENT LOOP (FILTERED)
xMain() {
  xInit
  xPrint atline 1 "Event filter demo. Press q or Ctrl-C to quit."
  while true; do zZ; xRawRead; done;
  xCleanup
}

xMain "$@"
# EOF
```
