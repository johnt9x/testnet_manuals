# Snapshot  nibiru (pruning: 100/0/19 | indexer: null | version tag: v0.19.2)

sudo systemctl stop nibid
```
cp $HOME/.nibid/data/priv_validator_state.json $HOME/.nibid/priv_validator_state.json.backup
```
rm -rf $HOME/.nolus/data
```
# Download latest snapshot

curl -O http://86.48.16.205/nibi/nibiru-itn-1_snapshot_latest.tar.lz4
```
tar -I lz4 -xf nibiru-itn-1_snapshot_latest.tar.lz4 -C $HOME/.nibid
```
mv $HOME/.nibid/priv_validator_state.json.backup $HOME/.nibid/data/priv_validator_state.json
```
# Restart the service and check the log

sudo systemctl start nibid && sudo journalctl -u nibid -f --no-hostname -o cat
```
