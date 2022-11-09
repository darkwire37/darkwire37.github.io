# Linux Distractions

## Console Trolls
### NMS:  (https://github.com/bartobri/no-more-secrets)
By setting a user's shell to the following shell script, every command they run will be "encrypted/decrypted" before the output is shown:
```
#!/bin/sh
while :
do
        echo -n $(pwd)
        echo -n "$"
        read cmd
        if [[ cmd =~ "cd". ]]
        then
                $cmd
        fi
        $cmd | nms -a -s
done
```
The output looks like:
```
$ ls -al
dx▀x¼Æb≈╔x└Σ0 .╗B6*ot%╪e;AΦúÅ¢ivVú6á∞4\9kº:cèsσOwöΩ√RV√╫
+║⌠í⌠?╧r_.ïh3┌ª«═▀' ≤ⁿr«X░o╬ñë╚ yySπß4╙═┘⌡ß»-¿T1í«5á6ï2φ∙
uwwxàwqÆ╧ZK±├(y≥α≤ó│t─(e▌!<½┴¼i$+c╤$ 'ªë≡¶u9ëhQß∞╩Ñú╗l±πa╕$═-⌐µ╧]Y▓\;«T┼3°┘╛Mr╧┌oß═⌠÷═
φe<FvuzVwT~τw[U┼▀╤îεt╨ueφöP₧⌡íG│÷a:πw╠≥üê%±£╕≤≤6≡╛3xx░j.vzup_⌡≥>q⌠VQ═ì7┤à<àσ0/NrsWΦ/çE═╟ƒº
├1░│$-╧m$■«_φT+<f⌐Z¼'╗╬N3yΦPª"5PåτφÇH4αJï┘£ùùà*\h═~Q┌0kRbé<úQ├&┘╪µd.
&π\y├ΦΩ--L╟=óªTófDtY╒≡{/è&⌡¼ùh▒=ïÅ╡}í┼0£₧ O2N≡öï<áû┴É╔s.ê⌠┌h┐qS
u╪╝i╪±-ö║¬¢ïD\∙îÄyVitEÇ█╫#4ïstΩê╬╟h╝╞Qv╕6òa¢δ δ├ ñÿI22Q%º£Ü▀╖
PöJx)l╘⌐ΣJz ≈k╘ub,å=(4╨à├∙V\DσzΣä┤É5*£û9r▒ÇvÉ≈╞µ∞â├Ü2àK.]πOv┼Z═'h
```
But it will fully "decrypt" over a few seconds

### Backwrd
