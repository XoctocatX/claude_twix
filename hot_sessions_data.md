Prompt:                                                                                                         
                                                   
  Настрой хуки для автосохранения "горячего контекста" между сессиями Claude Code. После каждого ответа ассистента
   — дописывать последний user prompt + последний текст assistant из transcript в файл
  <project-memory-dir>/hot-context.md с ротацией; при старте новой сессии — инжектить последние 120 строк этого   
  файла в additionalContext.                       
                                                                                                                  
  Требования:                                  
  1. Создай скрипт ~/.claude/hot-context-hook.sh со следующим содержимым (сделай исполняемым):                    
                                                                                                                 
  #!/usr/bin/env bash                                                                                             
  # hot-context-hook.sh — persist hot conversation context across sessions                                        
  set -euo pipefail                                                                                               
  MODE="${1:-write}"                                                                                              
  INPUT="$(cat)"                                                                                                  
  TRANSCRIPT=$(printf '%s' "$INPUT" | jq -r '.transcript_path // empty' 2>/dev/null || echo "")                   
  SESSION_ID=$(printf '%s' "$INPUT" | jq -r '.session_id // empty' 2>/dev/null || echo "")                        
  CWD_FROM_INPUT=$(printf '%s' "$INPUT" | jq -r '.cwd // empty' 2>/dev/null || echo "")
  CWD="${CWD_FROM_INPUT:-$PWD}"                                                                                   
  SANITIZED=$(printf '%s' "$CWD" | sed 's|^/||; s|_|-|g; s|/|-|g; s|^|-|')
  MEM_DIR="$HOME/.claude/projects/$SANITIZED/memory"
  HOT_CTX="$MEM_DIR/hot-context.md"             
  mkdir -p "$MEM_DIR"                                                                                             
                                                
  if [[ "$MODE" == "read" ]]; then                                                                                
    if [[ -f "$HOT_CTX" ]]; then                
      CONTENT=$(tail -120 "$HOT_CTX")                                                                             
      jq -n --arg ctx "$CONTENT" '{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:("# Hot 
  context from previous sessions\n\n" + $ctx)}}'                                                                  
    fi                                                                                                            
    exit 0                                                                                                        
  fi                                                                                                              
                                                                                                                  
  if [[ -z "$TRANSCRIPT" || ! -f "$TRANSCRIPT" ]]; then                                                           
    [[ -n "$SESSION_ID" ]] && TRANSCRIPT="$HOME/.claude/projects/$SANITIZED/$SESSION_ID.jsonl"
  fi                                                                                                              
  [[ ! -f "$TRANSCRIPT" ]] && exit 0            
                                                                                                                  
  LAST_USER=$(jq -rs '[.[] | select(.type == "user" and (.message.content | type == "string"))] | last | 
  .message.content // ""' "$TRANSCRIPT" 2>/dev/null | head -c 600 || echo "")                                     
  LAST_ASSISTANT=$(jq -rs '[.[] | select(.type == "assistant" and (.message.content | type == "array")) |         
  .message.content | [.[] | select(.type == "text") | .text] | join("\n") | select(length > 0)] | last // ""'     
  "$TRANSCRIPT" 2>/dev/null | head -c 800 || echo "")                                                             
  [[ -z "$LAST_USER$LAST_ASSISTANT" ]] && exit 0                                                                  
                                                
  TS=$(date +"%Y-%m-%d %H:%M MSK")          
  {                                                                                                               
    printf '\n## %s\n' "$TS"               
    printf '**USER:** %s\n\n' "$LAST_USER"                                                                        
    printf '**CLAUDE:** %s\n' "$LAST_ASSISTANT"                                                                   
    printf -- '---\n'                                                                                             
  } >> "$HOT_CTX"                                                                                                 
                                          
  if [[ $(wc -l < "$HOT_CTX") -gt 500 ]]; then                                                                    
    tail -500 "$HOT_CTX" > "$HOT_CTX.tmp" && mv "$HOT_CTX.tmp" "$HOT_CTX"                                         
  fi                                        
                                                                                                                  
  2. В ~/.claude/settings.json добавь (МЕРДЖИ с существующими ключами, не перезаписывай!):
                                                                                                                  
  {                                        
    "hooks": {                                                                                                    
      "Stop": [                                                                                                   
        {"hooks": [{"type":"command","command":"/Users/<USER>/.claude/hot-context-hook.sh write 2>/dev/null || 
  true","timeout":10}]}                                                                                           
      ],                                   
      "SessionStart": [                                                                                           
        {"hooks": [{"type":"command","command":"/Users/<USER>/.claude/hot-context-hook.sh read 2>/dev/null || 
  true","timeout":5}]}                                                                                            
      ]                                                                                                           
    }                                     
  }                                                                                                               
                                                
  Замени <USER> на реальный путь (используй $HOME через echo $HOME).                                              
                                            
  3. Проверь: pipe-test скрипт на последнем .jsonl в текущем проекте (ls -t ~/.claude/projects/*/*.jsonl | head 
  -1), подай на stdin {"transcript_path":"<path>","cwd":"<pwd>"} в режиме write и проверь что создан
  hot-context.md. Затем jq -e валидация hooks в settings.json.                                                    
  4. Скажи мне: чтобы хуки активировались в текущей сессии — откройте /hooks один раз или перезапустите Claude
  Code (watcher вотчит только директории с settings.json на момент старта сессии).   
