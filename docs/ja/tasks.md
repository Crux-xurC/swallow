# swallow のタスク

tezcatl の agenda で管理する表です。下の表を tezcatl で開き、`:w replace #eigenvalue tasks` で印をIDに置き換えてから、
`task_table take` で取り込み（markdown パネルなら表の上で `t`）、task_table で `L` を押すと agenda（`C-c a`）に載ります。
agenda での変更は task_table の book に戻り、この文書へは task_table の `W` で戻します。

tezcatl のファイルの起点（`~/my_app`）から見たこの文書のパス：`no_nix/swallow/docs/ja/tasks.md`

Table: swallow_#eigenvalue

| id   | state | pri | task                                                                        | inner_id    |
|------|-------|-----|-----------------------------------------------------------------------------|-------------|
| t-01 | DONE  | A   | 設計の叩き台を書く（design-memo.md）                                        | #eigenvalue |
| t-02 | DONE  | A   | 書く言語：素朴な版は python3（Citlalpol と揃える）       | #eigenvalue |
| t-03 | DONE  | A   | citld への結果の口（fd 3 か citld の socket か）を Citlalpol と一緒に決める | #eigenvalue |
| t-04 | DONE  | A   | S1：serve のデフォルト（HTTP/1.1、captured を記録して 501、秘密を伏せる）   | #eigenvalue |
| t-05 | TODO  | B   | match に書けるものと reply の形を決める                                     | #eigenvalue |
| t-06 | DONE  | B   | S2：照合と応答、swallow draft                                               | #eigenvalue |
| t-07 | TODO  | B   | fire の時機（応答の前か後か）を決める                                       | #eigenvalue |
| t-08 | TODO  | B   | S3：fire                                                                    | #eigenvalue |
| t-09 | TODO  | C   | S4：verify（safe / never / elsewhere、比べるところ）                        | #eigenvalue |
| t-10 | TODO  | C   | S5：TLS（Citlalpol M4 と）、HTTP 以外、export pact                          | #eigenvalue |
| t-11 | TODO  | C   | ライセンスを決める                                                          | #eigenvalue |
