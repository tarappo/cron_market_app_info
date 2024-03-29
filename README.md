# Example market_app_info
see: https://github.com/tarappo/market_app_info

## Usage Examples

```
jobs:
  ios_app_matrix:
    runs-on: ubuntu-latest
    strategy:
        matrix:
            bundle_identifier: ["com.facebook.Facebook", "com.atebits.Tweetie2"]
    steps:
      - uses: actions/checkout@v4
      - name: Generate ID
        id: build-id
        run: |
            echo "name=id::$(date +%s)" >> $GITHUB_OUTPUT
      - name: Cache
        id: cache
        uses: actions/cache@v3
        with:
            path: ${{ matrix.bundle_identifier }}_version.txt
            key: version-${{ steps.build-id.outputs.id }}
            restore-keys: version-
      - name: Current Version
        id: current_version
        run: |
            if [ -e ${{ matrix.bundle_identifier }}_version.txt ]; then
                echo "version.txtが存在します：$(cat ${{ matrix.bundle_identifier }}_version.txt)"
                if [ $(cat ${{ matrix.bundle_identifier }}_version.txt) = "null" ]
                then
                    echo "verion.txtの値がnullのため、0.0.0を出力します"
                    echo "version=0.0.0" >> $GITHUB_OUTPUT
                else
                    echo "verion.txtの値を出力します"
                    echo "version=$(cat ${{ matrix.bundle_identifier }}_version.txt)" >> $GITHUB_OUTPUT
                fi
            else
                echo "version=0.0.0"
            fi
      - name: Check Version
        run: |
            echo ${{ steps.current_version.outputs.version }}
      - name: iOS Market Information
        id: appstore_information
        uses: tarappo/market_app_info@main
        with:
            bundle_identifier: ${{ matrix.bundle_identifier }}
            version_number: ${{ steps.current_version.outputs.version }}
      - name: Latest Market Information Ouputs
        id: latest_output
        run: |
            echo ${{ steps.appstore_information.outputs.app_name }}
            echo ${{ steps.appstore_information.outputs.release_formatted_date }}
            echo ${{ steps.appstore_information.outputs.release_date }}
            echo ${{ steps.appstore_information.outputs.version }}

            # 値がある場合は、version.txtを更新する
            if [ "${{ steps.appstore_information.outputs.version }}" != "null" ]; then
                echo "version.txtを更新します：${{ steps.appstore_information.outputs.version }}"
                echo ${{ steps.appstore_information.outputs.version }} > ${{ matrix.bundle_identifier }}_version.txt
                echo "new_version=$(cat ${{ matrix.bundle_identifier }}_version.txt)" >> $GITHUB_OUTPUT
            else
                echo "最新バージョンはありませんでした"
                echo "new_version=null" >> $GITHUB_OUTPUT
            fi
      - name: Post to a Slack channel
        if: ${{ steps.latest_output.outputs.new_version != 'null' }}
        id: slack
        uses: slackapi/slack-github-action@v1.24.0
        with:
            payload: |
                {
                "text": "対象アプリ：${{ steps.appstore_information.outputs.app_name }}\n最新バージョン：${{ steps.latest_output.outputs.new_version }}（${{ steps.appstore_information.outputs.release_formatted_date }}）",
                "blocks": [
                    {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": "対象アプリ：${{ steps.appstore_information.outputs.app_name }}\n最新バージョン：${{ steps.latest_output.outputs.new_version }}"
                    }
                    }
                ]
                }
        env:
            SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

```


