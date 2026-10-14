# V153_4__invalidate_reward_settlement.sql — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/resources/db/postgresql/V153_4__invalidate_reward_settlement.sql`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–61행


````sql
-- 정산 원본 변경만 회차 버전을 올리고 계산 결과 저장은 무효화에서 제외한다.
CREATE FUNCTION invalidate_reward_settlement() RETURNS trigger LANGUAGE plpgsql AS $$
DECLARE
    previous_source jsonb;
    current_source jsonb;
    owner_id bigint;
BEGIN
    IF TG_TABLE_NAME = 'reward_cycle' THEN
        IF ROW(OLD.user_id, OLD.starts_at, OLD.ends_at, OLD.maximum_won,
               OLD.eligible_days, OLD.calculation_denominator, OLD.policy_version,
               OLD.eligibility, OLD.review_reason)
           IS DISTINCT FROM
           ROW(NEW.user_id, NEW.starts_at, NEW.ends_at, NEW.maximum_won,
               NEW.eligible_days, NEW.calculation_denominator, NEW.policy_version,
               NEW.eligibility, NEW.review_reason) THEN
            NEW.input_revision := OLD.input_revision + 1;
            NEW.next_settlement_at := LEAST(COALESCE(NEW.next_settlement_at, CURRENT_TIMESTAMP), CURRENT_TIMESTAMP);
        END IF;
        RETURN NEW;
    END IF;

    previous_source := CASE WHEN TG_OP <> 'INSERT' THEN to_jsonb(OLD) END;
    current_source := CASE WHEN TG_OP <> 'DELETE' THEN to_jsonb(NEW) END;
    IF TG_TABLE_NAME = 'reward_learning_fact' THEN
        previous_source := previous_source - ARRAY['cycle_id', 'earned_won', 'balance_won', 'calculation_version', 'response_payload'];
        current_source := current_source - ARRAY['cycle_id', 'earned_won', 'balance_won', 'calculation_version', 'response_payload'];
    ELSE
        previous_source := previous_source - 'captured_at';
        current_source := current_source - 'captured_at';
    END IF;
    IF previous_source IS NOT DISTINCT FROM current_source THEN RETURN NULL; END IF;

    FOR owner_id IN
        SELECT DISTINCT value FROM unnest(ARRAY[
            (previous_source->>'user_id')::bigint, (current_source->>'user_id')::bigint
        ]) AS owners(value) WHERE value IS NOT NULL ORDER BY value
    LOOP
        UPDATE reward_cycle SET input_revision = input_revision + 1,
            next_settlement_at = LEAST(COALESCE(next_settlement_at, CURRENT_TIMESTAMP), CURRENT_TIMESTAMP)
        WHERE user_id = owner_id AND EXISTS (
            SELECT 1 FROM jsonb_array_elements(jsonb_build_array(previous_source, current_source)) source
            WHERE (source->>'user_id')::bigint = owner_id AND
                CASE WHEN TG_TABLE_NAME = 'reward_learning_fact' THEN
                    starts_at <= (source->>'completed_at')::timestamptz
                    AND ends_at > (source->>'completed_at')::timestamptz
                ELSE
                    starts_at < (((source->>'learning_date')::date + 1)::timestamp AT TIME ZONE 'Asia/Seoul')
                    AND ends_at > ((source->>'learning_date')::date::timestamp AT TIME ZONE 'Asia/Seoul')
                END
        );
    END LOOP;
    RETURN NULL;
END;
$$;

CREATE TRIGGER reward_cycle_input_revision BEFORE UPDATE ON reward_cycle
FOR EACH ROW EXECUTE FUNCTION invalidate_reward_settlement();
CREATE TRIGGER reward_fact_input_revision AFTER INSERT OR UPDATE OR DELETE ON reward_learning_fact
FOR EACH ROW EXECUTE FUNCTION invalidate_reward_settlement();
CREATE TRIGGER reward_assignment_input_revision AFTER INSERT OR UPDATE OR DELETE ON reward_daily_assignment
FOR EACH ROW EXECUTE FUNCTION invalidate_reward_settlement();
````
