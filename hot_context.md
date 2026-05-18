Когда обновлять                                                                                               
                                                                                                                  
    После каждого ответа, где произошло material state change — ДО отправки ответа, не батчем в конце сессии.     
                                                                                                                  
    Триггеры:                                                                                                     
    - Deploy completed (Worker version + release_sha, Modal version + commit, Dashboard preview URL, public docs  
    commit)                                                                                                       
    - PR merge (squash SHA, что теперь в main vs что pending)                                                     
    - Component status flip (audit phase done, task closed, P0 fix shipped)                                       
    - Queue / priority decision (next default, blocked-by, deferred items)                                        
    - Worktree create/remove (location + purpose + cleanup ETA)                                                   
    - Material findings (P0 promotion, new audit gap, decision with future weight)                                
                                                                                                                  
    Skip:                                                                                                         
    - Routine progress logs / 5-min validation snapshots                                                          
    - Exploratory commands без conclusion                                                                         
    - Уже видно из git log / current .env / live state                                                            
    - Уже задокументировано в CLAUDE.md                                                                           
                                                                                                                  
    Куда писать                                                                                                   
                                                                                                                  
    ┌─────────────────────────────────────────────┬────────────────────────────────────────────────────────────── 
  ──┐                                                                                                             
    │                  артефакт                   │                           назначение                          
    │                                                                                                             
    ├─────────────────────────────────────────────┼────────────────────────────────────────────────────────────── 
  ──┤                                                                                                             
    │ MEMORY.md → Current Status → TODAY HOT      │ Главное место — first thing next-me reads. Live deploy SHAs,  
    │                                                                                                             
    │ POINTERS block                              │ version IDs, migration numbers, current task, open findings.  
    │                                                                                                             
    ├─────────────────────────────────────────────┼────────────────────────────────────────────────────────────── 
  ──┤                                                                                                             
    │ memory/post_56_priority_queue.md (или       │ task ordering, blocked-by chain                               
    │                                                                                                             
    │ текущий queue file)                         │                                                               
    │                                                                                                             
    ├─────────────────────────────────────────────┼────────────────────────────────────────────────────────────── 
  ──┤                                                                                                             
    │ Topic files                                 │ стабильные знания (architecture, APIs, conventions)           
    │                                                                                                             
    ├─────────────────────────────────────────────┼────────────────────────────────────────────────────────────── 
  ──┤                                                                                                             
    │ Новый feedback file                         │ persistent user preferences                                   
    │                                                                                                             
    └─────────────────────────────────────────────┴────────────────────────────────────────────────────────────── 
  ──┘                                                                                                             
                                                                                                                  
    Operational pattern (каждый turn)                                                                             
                                                                                                                  
    1. Делаю работу что просил user                                                                               
    2. До составления финального ответа — scan: что material изменилось? (deploy / merge / decision / status /    
    location)                                                                                                     
    3. Если да — Edit memory inline (MEMORY.md TODAY HOT POINTERS + queue/topic files)                            
    4. Отправляю ответ. Memory уже consistent.                                                                    
                                                                                                                  
    Почему                                                                                                        
                                                                                                                  
    Hot-context-hook lossy на длинных technical turn'ах. Сессии могут оборваться (reboot, network drop, /clear, compaction). Persisted memory files — единственный надёжный hand-off. При разрыве — next-me читает MEMORY.md 
  и сразу знает: main SHA, версии всех компонентов, текущая задача, открытые findings. Без "что было сделано / что потеряно"..
